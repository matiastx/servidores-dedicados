# ServidoresDedicados

Servidores de juegos dedicados, desplegados con **Coolify** sobre el VPS `server`
(`192.168.68.222` LAN / `100.75.207.93` tailnet / `181.164.89.26` publica via NAT).

| Server | Imagen | Compose | Puertos |
|---|---|---|---|
| Project Zomboid | `danixu86/project-zomboid-dedicated-server` | `zomboid/docker-compose.yml` | UDP `16261`, `16262`, `8766`, `8767` · TCP `16262-16272` |

## Estructura

```
zomboid/
  docker-compose.yml   # servicio, puertos y variables que consume Coolify
  .env.example         # referencia de variables (los secretos viven en Coolify)
  README.md            # runbook del server (admin, mods, updates, backup)
```

## Deploy en Coolify

1. Proyecto **ServidoresDedicados** → ambiente `production`.
2. Nueva app → **Docker Compose** (repo público `matiastx/servidores-dedicados`).
   - Base Directory: `/zomboid`
   - Docker Compose Location: `/docker-compose.yml` (es **relativa** al base
     directory: Coolify resuelve `./{base_directory}{docker_compose_location}`)
   - Server: `localhost` (el VPS donde corre Coolify)
3. **Environment Variables**: copiar `.env.example` y setearlas en el panel. Los
   secretos (`RCONPASSWORD`) se marcan como *shown once*. `ADMINPASSWORD` solo
   hace falta para el primer arranque que crea el mundo (ver `zomboid/README.md`).
4. **Persistent Volumes**: en apps Docker Compose **no** se define nada en el
   panel — los volumes van declarados en el compose. Coolify los parsea, los
   renombra a `{uuid_app}_{nombre}` y los materializa como volumes Docker:
   - `zomboid-data` → `/home/steam/Zomboid` (mundo + configuración)
   - `zomboid-workshop` → `/home/steam/pz-dedicated/steamapps/workshop` (mods)
5. Deploy.

La app **no** lleva dominio: no es HTTP, se accede por `IP:puerto` desde el juego.

> **Límites de recursos**: en apps Docker Compose los campos *Memory* / *CPU* del
> panel no se aplican. Los límites van en el compose (`mem_limit`, `cpus`).

## Exposición

El VPS está detrás de NAT (la IP pública `181.164.89.26` es del router `192.168.68.1`),
así que hay que hacer **port forwarding en el router** hacia `192.168.68.222`:

| Protocolo | Puerto externo | Destino | Necesario |
|---|---|---|---|
| UDP | 16261 | 192.168.68.222:16261 | **si** (puerto principal) |
| UDP | 16262 | 192.168.68.222:16262 | **si** (direct connection) |
| TCP | 16262-16272 | 192.168.68.222:16262-16272 | por requerimiento (slots) |
| UDP | 8766 | 192.168.68.222:8766 | opcional (Steam legacy) |
| UDP | 8767 | 192.168.68.222:8767 | opcional (Steam legacy) |

- **UDP 16261 + UDP 16262** son los unicos puertos que hacen falta para que un
  jugador entre por IP. En Build 42 el trafico de juego va por UDP: el server
  abre el socket UDP 16261 y el 16262 es el "direct connection port".
- **TCP 16262-16272** son el rango de slots (8 configurados, `MaxPlayers=8`).
  Se mantienen abiertos por requerimiento y por compatibilidad.
- **UDP 8766/8767** los usa el cliente de Steam (legacy).
- **TCP 27015** (RCON) esta publicado **solo en loopback** (`127.0.0.1:27015`),
  para el bot que corre en el host: **no** hacer port forwarding ni publicarlo
  en la LAN.

Cloudflare no proxea UDP de juegos: la conexión va por IP directa.
Dentro de la LAN/Tailscale el server también es accesible en `192.168.68.222:16261`
y `100.75.207.93:16261`.

## Actualizar la versión del juego

La imagen trae el juego horneado y el entrypoint **no** actualiza salvo
`FORCEUPDATE=true`. Para mover la versión: cambiar el tag de `image:` en
`zomboid/docker-compose.yml`, commit, y redeploy desde Coolify.
