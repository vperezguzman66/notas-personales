---
proyecto: "hosting-gestionado"
ruta: "sin repo — proyecto de infraestructura/ops"
cliente: "Propio (piloto de nuevo servicio VP Services IT)"
stack: "Hetzner dedicado + Proxmox VE + Debian 12 + Docker + Caddy + Cloudflare"
estado: "Piloto — checklist Semana 1 definido, aún no ejecutado"
ultimo_cambio: 2026-07-12
---

[[Varios]]

Piloto de negocio "Hosting Gestionado" bajo VP Services IT: ofrecer VMs gestionadas a clientes (pymes) sobre un servidor dedicado propio, con onboarding rápido (clonar una plantilla endurecida por cliente en menos de 10 minutos). No es un proyecto de código — es infraestructura/ops. La idea y el checklist técnico se armaron en una conversación de Claude por app móvil (no Claude Code), y el checklist se guardó como `~/Downloads/checklist-semana1-hosting-gestionado.md`.

## Arquitectura elegida

- **Proveedor**: Hetzner, servidor dedicado línea **AX41/AX42** (Ryzen, 64 GB RAM, 2x NVMe) — también se evalúa el Server Auction de Hetzner (servidores usados, más baratos) como alternativa.
- **Datacenter**: Falkenstein (FSN1) o Helsinki — indiferente en latencia para clientes en Chile.
- **Hipervisor**: Proxmox VE sobre Debian 12, instalado vía Rescue System + `installimage`, con **RAID1 por software (mdadm)** sobre los 2 NVMe.
- **Red**: bridge `vmbr0` con NAT interno (10.10.10.0/24) — evita comprar IPs adicionales; cada servicio de cliente sale por proxy inverso (Caddy) con SSL automático vía Let's Encrypt.
- **Plantilla de cliente**: VM Debian 12 endurecida (SSH solo por llave, ufw, fail2ban, unattended-upgrades, zona horaria America/Santiago) con Docker + Docker Compose (repo oficial de Docker) y estructura estándar:
  ```
  /srv/apps/          → docker-compose por servicio
  /srv/apps/proxy/    → Caddy o Traefik
  /srv/backups/       → staging local de respaldos
  ```
  Se limpia (historial, host keys SSH) y se convierte en **Template** de Proxmox para clonar por cliente.
- **DNS/proxy externo**: Cloudflare (plan gratis) para apuntar subdominios de clientes al servidor.

## Checklist Semana 1 (piloto técnico)

Objetivo: servidor Hetzner con Proxmox VE operativo + plantilla Debian endurecida con Docker, lista para clonar por cliente.

- **Día 1 — Contratación y acceso**: cuenta Hetzner, contratar AX41/AX42, elegir datacenter, SSH key dedicada, activar Rescue System, guardar credenciales en vault (Bitwarden/KeePass).
- **Día 2 — Instalación de Proxmox VE**: `installimage` con Debian 12 mínimo + RAID1 mdadm (particionado: `/boot` 1GB, `swap` 8GB, `/` 40GB, resto LVM), agregar repo `pve-no-subscription`, instalar `proxmox-ve`, verificar UI en `https://IP:8006`, crear bridge `vmbr0`.
- **Día 3 — Seguridad del hipervisor**: deshabilitar login root/password por SSH, usuario admin propio con sudo, firewall Proxmox (default DROP, solo SSH + 8006 desde IP propia), fail2ban, 2FA TOTP en la UI, unattended-upgrades, WireGuard opcional para cerrar 8006 al mundo.
- **Día 4 — Plantilla Debian endurecida**: VM base (2 vCPU, 4GB RAM, 40GB disco, qemu-guest-agent), hardening completo, Docker oficial, estructura `/srv/apps/`, Caddy como proxy inverso, limpiar host keys y convertir a Template.
- **Día 5 — Validación y cierre**: clonar plantilla → VM demo, verificar arranque/red/Docker/Caddy+SSL, probar snapshot/rollback, apuntar subdominio de prueba (demo.vpservices-it.com) vía Cloudflare, documentar en `NOTAS-SETUP.md`, medir consumo base del host para dimensionar cuántos clientes caben.

**Criterio de éxito**: crear una VM nueva para un cliente en menos de 10 minutos clonando la plantilla, ya segura, con Docker y proxy SSL listos.

## Semana 2 (pendiente — bloqueante)

Respaldos automatizados (restic → Backblaze B2, regla 3-2-1) y monitoreo (Uptime Kuma + alertas). **No se acepta ningún cliente real antes de completar esto.**

## Costos (referencia julio 2026)

| Ítem | Costo aprox. |
|---|---|
| Servidor Hetzner AX41 | ~€46/mes |
| IPs adicionales | Evitadas (NAT + proxy inverso) |
| Cloudflare | Gratis (plan Free) |
| **Total Semana 1** | **~€50/mes ≈ 1,4 UF** |

## Estado (2026-07-12)

Checklist definido, pendiente de ejecución — aún no se contrató el servidor Hetzner. Próximo paso: Día 1 (contratar servidor + acceso SSH).
