# Lab SSO self-hosted — Keycloak + oauth2-proxy

Laboratorio de **single sign-on** completamente autoalojado en Docker Compose sobre una VM Ubuntu Server: un IdP central (Keycloak) dando autenticación tanto a aplicaciones **con OIDC nativo** (Nextcloud) como a aplicaciones **sin él** (GLPI, protegido mediante oauth2-proxy), todo detrás de un Nginx con TLS.

Este lab fue la base sobre la que después implanté SSO con Keycloak en un entorno real de la administración pública durante mis prácticas.

---

## Arquitectura

```
            Navegador (https://*.lab)
                      │
              ┌───────▼────────┐
              │     Nginx      │  TLS + proxy inverso
              │ (resolver 127. │  keycloak.lab / nextcloud.lab / glpi.lab
              │  0.0.11, vars) │
              └───┬───────┬────┘
                  │       │
        ┌─────────▼──┐  ┌─▼────────────┐
        │  Keycloak  │  │ oauth2-proxy │──► GLPI (header auth)
        │ (realm:    │  └──────────────┘
        │  demo)     │        Nextcloud (OIDC nativo)
        └────┬───────┘
             │
   PostgreSQL (Keycloak y Nextcloud) · MariaDB (GLPI)
```

**Flujo SSO sobre GLPI:** petición a `glpi.lab` → oauth2-proxy intercepta → redirección a Keycloak (realm `demo`, OIDC) → login → callback → oauth2-proxy inyecta la identidad vía cabecera (`REMOTE_USER`) → GLPI autentica al usuario sin pedir credenciales propias.

## Servicios

| Servicio | Imagen | Función |
|---|---|---|
| Keycloak | `quay.io/keycloak/keycloak:26.x` | Identity Provider (OIDC) |
| oauth2-proxy | `quay.io/oauth2-proxy/oauth2-proxy` | Gateway de autenticación para apps sin OIDC |
| GLPI | `glpi/glpi:10.0` | Helpdesk / inventario (app protegida) |
| Nextcloud | `nextcloud:33-apache` | Nube de archivos (OIDC nativo) |
| Nginx | `nginx:alpine` | Proxy inverso + terminación TLS |
| PostgreSQL ×2 | `postgres:16-alpine` | BBDD de Keycloak y Nextcloud |
| MariaDB | `mariadb:11.4` | BBDD de GLPI |

<!-- TODO: sube aquí el docker-compose.yml y la carpeta nginx/ cuando reconstruyas el lab -->
📌 *El `docker-compose.yml` está en proceso de reconstrucción y se subirá al repo.*

---

## Problemas reales resueltos

Lo interesante del lab no fue levantarlo, fue todo lo que se rompió por el camino:

1. **Resolución de nombres dentro de Docker.** Las URLs de OIDC (`keycloak.lab`) deben resolver igual desde el navegador y desde dentro de los contenedores. Solución: eliminar `extra_hosts` con IPs hardcodeadas (que cambian en cada recreación) y usar **aliases de red** sobre el contenedor de Nginx.
2. **Nginx no arrancaba si un upstream estaba caído.** Con nombres estáticos en `proxy_pass`, Nginx resuelve en el arranque y muere si falta un contenedor. Solución: `resolver 127.0.0.11` (DNS interno de Docker) y upstreams como **variables** (`set $upstream ...`) para resolución en tiempo de petición.
3. **Deadlock de arranque entre Nginx y oauth2-proxy**, cada uno esperando al otro. Resuelto reordenando dependencias y la resolución dinámica del punto anterior.
4. **Cookies CSRF de oauth2-proxy fallando en el flujo de login.** Resuelto con `--cookie-samesite=none`, `--cookie-csrf-per-request=true` y `--cookie-csrf-expire=5m`.
5. **Permisos de volúmenes de GLPI** impidiendo el arranque. Resuelto ejecutando el ajuste como root con `docker compose run --user 0`.
6. **Variables de entorno de GLPI mal documentadas.** Leyendo el *entrypoint* de la imagen se confirmó que son `GLPI_DB_*` y no `GLPI_DATABASE_*`.
7. **El dominio `.local` rompe en Windows** (lo trata como mDNS/Bonjour y el navegador no aplica bien las excepciones de proxy). Migración completa del lab a `.lab`.
8. **Proxy corporativo (Squid) interceptando el tráfico** hacia el lab. Resuelto añadiendo `*.lab` y la IP de la VM a las excepciones de proxy del sistema.

---

## Pendiente / siguientes pasos

- Re-subir `docker-compose.yml` y configuración de Nginx.
- Afinar la autenticación por cabeceras en GLPI (campo `REMOTE_USER`, URL de logout hacia `/oauth2/sign_out`).
- `trusted_domains` de Nextcloud vía `occ`.
- Probar identity brokering de Keycloak con un IdP externo.

---

*Stack: Docker Compose · Keycloak · oauth2-proxy · OIDC · Nginx · GLPI · Nextcloud · PostgreSQL · MariaDB · Ubuntu Server*
