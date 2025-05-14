## Blog Post: Securing CryptPad Behind Traefik via Portainer

In diesem Beitrag schildere ich den teils holprigen Weg, **CryptPad** hinter **Traefik v3** mit **Cloudflare Full (Strict)** SSL abzusichern. Die Learnings rund um Zertifikate und Content-Security-Policy (CSP) haben mir gezeigt, wie komplex das Zusammenspiel aus Reverse-Proxy, Browser-Security und Docker-Umgebung sein kann.

---

### 1. Ausgangssituation & erster Ansatz

* **Goal**: CryptPad auf `pad.example.com` (Haupt-Domain) und `pad-sandbox.example.com` (Sandbox) betreiben.
* **Voraussetzung**: Cloudflare Full (Strict) → Origin benötigt vertrauenswürdiges Zertifikat.
* **Deployment**: Docker-Compose über Portainer, Traefik als Reverse-Proxy.

Zunächst punktete Traefik mit selbsterstellten Zertifikaten, die Cloudflare **nicht** akzeptierte – typische **502 Bad Gateway**-Meldung. Hier startete die erste Learning-Curve: Traefik v3 trennt statische und dynamische TLS-Konfiguration komplett anders als v2.

---

### 2. TLS: Dynamischer Store statt entryPoints

**Trial & Error**: Anfänglich versuchten wir, TLS-Zertifikate direkt unter `entryPoints.https` zu definieren – Fehlermeldung `field not found, node: tls`. Nach Recherche im Traefik-Forum war klar:

* **Traefik v3** lädt Zertifikate ausschließlich über **dynamische** TLS-Stores.
* **entryPoints** bleiben clean, der `tls:`-Block muss in eine **dynamic**-Datei.

**Final Setup**:

1. **Wildcard-Zertifikate** (`*.example.com`) in `traefik-ssl-certs/`.
2. **`config/dynamic/tls.yml`** mit dem Default-Store:

   ```yaml
   tls:
     stores:
       default:
         defaultCertificate:
           certFile: /ssl-certs/wildcard.crt
           keyFile:  /ssl-certs/wildcard.key
   ```
3. **Statische** `traefik.yml` ohne TLS unter entryPoints:

   ```yaml
   entryPoints:
     http:
       address: ":80"
       http:
         redirections:
           entryPoint:
             to: https
             scheme: https
     https:
       address: ":443"
   ```
4. Traefik neu deployen – **Cloudflare Full (Strict)** nimmt jetzt das Origin-Zertifikat an.

---

### 3. Traefik-Labels in Portainer & ein Router

Nach dem TLS-Durchbruch stellte sich die Frage: Einen Router für jede Domain? Zu kompliziert. **Lösung**: Ein einziger Router in Docker-Compose-Labels, der beide Hosts abdeckt:

```yaml
services:
  cryptpad:
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=frontend"
      - "traefik.http.services.cryptpad.loadbalancer.server.port=3000"
      - "traefik.http.routers.cryptpad.entrypoints=https"
      - "traefik.http.routers.cryptpad.rule=Host(`pad.example.com`) || Host(`pad-sandbox.example.com`)"
      - "traefik.http.routers.cryptpad.tls=true"
```

Damit entfiel *mehrmaliges* Chaos mit Pfadpräfixen und Eval-Policies – CryptPad kümmert sich intern um seine CSP-Varianten.

---

### 4. COEP/COOP fürs Sandbox-Iframe

Der nächste Stolperstein: **Cross-Origin-Embedder-/Opener-Policy**. Ohne sie blockierte Firefox das Sandbox-Iframe mit `NS_ERROR_DOM_COEP_FAILED`. Wieder half eine Inline-Middleware:

```yaml
- "traefik.http.routers.cryptpad.middlewares=cryptpad-coep"
- "traefik.http.middlewares.cryptpad-coep.headers.customResponseHeaders.Cross-Origin-Embedder-Policy=require-corp"
- "traefik.http.middlewares.cryptpad-coep.headers.customResponseHeaders.Cross-Origin-Opener-Policy=same-origin"
```

Einfach in die Labels gehängt, Iframe lädt, Fehler weg.

---

### 5. CSP-Warnungen verstehen und akzeptieren

Am Ende blieben noch rote CSP-Warnungen in der Konsole:

```
blocked inline script…
eval panic…
```

Das **sind** aber **Selbsttests von CryptPad**:

* Inline-Skripte und `eval()` werden auf Nicht-Editor-Pfaden **absichtlich** blockiert.
* Das zeigt: CSP greift. Funktioniert alles.

Wir haben gelernt: **Nicht jede Warnung ist ein Fehler** – teilweise signalisieren sie korrekte Sicherheitsmechanismen.

---

## Fazit & Learnings

* **Traefik v3** TLS-Konfiguration braucht dynamische Stores, nicht entryPoints.
* Cloudflare Full (Strict) erfordert vertrauenswürdige Origin-Zertifikate (Wildcard oder ACME).
* Traefik-Labels in Portainer ermöglichen *deklariertes Chaos*: ein Router statt vieler.
* COEP/COOP für Sandbox-Iframes per Inline-Middleware lösen Browserfehler sofort.
* CSP-Warnungen von Anwendungen nicht blind unterdrücken – oft sind sie gewollt.

Die Lernkurve war hoch, insbesondere beim CSP-Part und der Traefik-Konfiguration. Am Ende läuft CryptPad sicher, performant und wartbar hinter Traefik & Cloudflare.

Happy Hosting!
