# dontforget
Here I want to store some important files, which are maybe useful for some else

# 🚀 CryptPad + OnlyOffice Docker Setup

This repository provides a Docker-based setup to self-host [CryptPad](https://cryptpad.org) along with [OnlyOffice Document Server](https://www.onlyoffice.com) for collaborative document editing.

> **Live Domains**  
> 📄 Main CryptPad: [https://pad.funnysite.com](https://pad.funnysite.com)  
> 🧪 Sandbox CryptPad: [https://pad-sandbox.funnysite.com](https://pad-sandbox.funnysite.com)

---

## 📁 Files Overview

### `docker-compose.yml`
Defines the service stack for:
- `cryptpad`: Main app container
- `onlyoffice-document-server`: Internal document editing service
- Named volumes for persistence
- Separate networks for secure traffic routing (frontend/backend)
- Traefik labels for HTTPS routing and middleware (COOP/COEP)

### `.env`
Environment configuration used by `docker-compose`. These variables define the domains, OnlyOffice integration, and secrets.

Example:
```env
CPAD_MAIN_DOMAIN=https://pad.funnysite.com
CPAD_SANDBOX_DOMAIN=https://pad-sandbox.funnysite.com
CPAD_CONF=/cryptpad/config/config.js
CPAD_ONLYOFFICE_URL=http://onlyoffice-document-server
CPAD_INSTALL_ONLYOFFICE=yes
ONLYOFFICE_JWT_SECRET=JWT_TOKEN
