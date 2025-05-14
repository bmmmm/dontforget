Blog Post: Securing CryptPad Behind Traefik via Portainer

This post walks through the somewhat bumpy journey of hosting CryptPad behind Traefik v3 using Portainer, with Cloudflare Full (Strict) SSL and the correct headers & CSP settings.

⸻

1. Initial Setup and First Attempts
	•	Goal: Serve CryptPad on pad.example.com (main domain) and pad-sandbox.example.com (sandbox) via Docker Compose managed in Portainer.
	•	Requirement: Cloudflare in Full (Strict) mode requires Traefik to present a trusted origin certificate.

Initially, Traefik generated self-signed certificates that Cloudflare refused, resulting in 502 Bad Gateway errors. This kicked off the first learning curve: Traefik v3 handles TLS configuration very differently compared to v2.

⸻

2. TLS Setup: Dynamic Store Instead of entryPoints

Trial & Error: I first tried defining TLS certificates directly under entryPoints.https in the static traefik.yml, which led to errors like field not found, node: tls. After digging through the Traefik documentation and community discussions, I learned that:
	•	Traefik v3 requires certificates to be loaded via dynamic TLS stores.
	•	The static entryPoints section should remain clean; all certificate definitions belong in dynamic files.

Final TLS Configuration:
	1.	Place your wildcard certificate files (*.example.com) into traefik-ssl-certs/:
	•	wildcard.crt
	•	wildcard.key
	2.	Create a dynamic TLS store in config/dynamic/tls.yml:

tls:
  stores:
    default:
      defaultCertificate:
        certFile: /ssl-certs/wildcard.crt
        keyFile:  /ssl-certs/wildcard.key


	3.	Keep your static traefik.yml free of TLS under entryPoints:

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


	4.	Redeploy Traefik in Portainer. Now, Cloudflare Full (Strict) successfully accepts the origin certificate.

⸻

3. Single Router for Both Domains

Instead of juggling multiple routers, I simplified the setup to use one Traefik router that matches both the main and sandbox domains. CryptPad itself handles internal CSP policies based on the request path.

services:
  cryptpad:
    image: cryptpad/cryptpad:latest
    env_file:
      - stack.env  # defines CPAD_MAIN_DOMAIN and CPAD_SANDBOX_DOMAIN
    volumes:
      - data:/cryptpad/data
      - /path/to/config.js:/cryptpad/config/config.js:ro
    networks:
      - frontend
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=frontend"
      - "traefik.http.services.cryptpad.loadbalancer.server.port=3000"
      - "traefik.http.routers.cryptpad.entrypoints=https"
      - "traefik.http.routers.cryptpad.rule=Host(`pad.example.com`) || Host(`pad-sandbox.example.com`)"
      - "traefik.http.routers.cryptpad.tls=true"

Mounting config.js ensures CryptPad loads the correct httpUnsafeOrigin and httpSafeOrigin values from your environment variables.

⸻

4. Adding COEP/COOP Headers for Sandbox Iframes

Modern browsers require proper Cross-Origin-Embedder Policy and Cross-Origin-Opener Policy headers for sandboxed iframes. Without them, Firefox throws NS_ERROR_DOM_COEP_FAILED. A simple inline middleware in the Traefik labels fixes this:

    # within the cryptpad service labels
    - "traefik.http.routers.cryptpad.middlewares=cryptpad-coep"
    - "traefik.http.middlewares.cryptpad-coep.headers.customResponseHeaders.Cross-Origin-Embedder-Policy=require-corp"
    - "traefik.http.middlewares.cryptpad-coep.headers.customResponseHeaders.Cross-Origin-Opener-Policy=same-origin"

After redeployment, the sandbox iframe loads without cross-origin errors.

⸻

5. Understanding CSP Warnings

You might still see red CSP warnings in the browser console:

blocked inline script…
eval panic…

These are self-tests built into CryptPad:
	•	It deliberately blocks inline scripts and eval() on non-editor paths to verify CSP enforcement.
	•	Seeing these warnings means your CSP is working as intended.

Instead of fighting every warning, recognize that they confirm the security policy is in place.

⸻

Conclusion & Key Takeaways
	1.	Traefik v3 TLS uses dynamic stores, not entryPoints.
	2.	Wildcard or ACME certificates must be loaded into the dynamic TLS store for Cloudflare Full (Strict).
	3.	A single router can match multiple domains, simplifying label configuration in Portainer.
	4.	COEP/COOP headers enable sandbox iframes to function correctly.
	5.	CSP warnings from CryptPad are often intentional self-tests.

Although the SSL and CSP parts involved significant trial and error, the result is a robust, secure, and maintainable setup for running CryptPad behind Traefik & Cloudflare.

Happy Hosting!