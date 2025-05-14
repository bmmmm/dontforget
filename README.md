# dontforget
Here I want to store some important files, which are maybe useful for some else

# 🚀 CryptPad + OnlyOffice Docker Setup

This repository provides a Docker-based setup to self-host [CryptPad](https://cryptpad.org) along with [OnlyOffice Document Server](https://www.onlyoffice.com) for collaborative document editing.

> **Live Domains**  
> 📄 Main CryptPad: [https://pad.example.com](https://pad.example.com)  
> 🧪 Sandbox CryptPad: [https://pad-sandbox.example.com](https://pad-sandbox.example.com)

---

## 📁 Files Overview

### `cryptpad onlyoffice traefik/docker-compose.yml`
Defines the service stack for:
- `cryptpad`: Main app container
- `onlyoffice-document-server`: Internal document editing service
- Named volumes for persistence
- Separate networks for secure traffic routing (frontend/backend)
- Traefik labels for HTTPS routing and middleware (COOP/COEP)

### `cryptpad onlyoffice traefik/.ENV`
Environment configuration used by `docker-compose`. These variables define the domains, OnlyOffice integration, and secrets.

Example:
```env
CPAD_MAIN_DOMAIN=https://pad.example.com
CPAD_SANDBOX_DOMAIN=https://pad-sandbox.example.com
CPAD_CONF=/cryptpad/config/config.js
CPAD_ONLYOFFICE_URL=http://onlyoffice-document-server
CPAD_INSTALL_ONLYOFFICE=yes
ONLYOFFICE_JWT_SECRET=JWT_TOKEN
```

### `cryptpad onlyoffice traefik/cryptpad-traefik-blogpost-de.md`
A blog post in German explaining the setup and deployment of CryptPad with Traefik.

### `cryptpad onlyoffice traefik/cryptpad-traefik-blogpost-en.md`
A blog post in English explaining the setup and deployment of CryptPad with Traefik.

---

## 🛠 Deployment with Portainer

This setup was deployed using [Portainer](https://www.portainer.io/) as a Docker Stack. Portainer provides an easy-to-use interface for managing Docker environments, including deploying stacks defined in `docker-compose.yml` files.

To deploy this stack in Portainer:
1. Navigate to the **Stacks** section in Portainer.
2. Click **Add Stack**.
3. Upload the `docker-compose.yml` file from the `cryptpad onlyoffice traefik/` folder.
4. Set the environment variables as defined in the `.ENV` file.
5. Deploy the stack.

### Note on `.ENV` File in Portainer
If you paste the content of the `.ENV` file directly into Portainer's **Advanced Mode** while deploying the stack, Portainer will automatically name it `.stack`. This eliminates the need to manually upload the `.ENV` file.
