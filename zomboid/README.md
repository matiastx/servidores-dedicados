# Server de Project Zomboid

Runbook operativo del server dedicado. Imagen:
[`danixu86/project-zomboid-dedicated-server`](https://github.com/Danixu/project-zomboid-server-docker).

## Arranque

El primer arranque crea el mundo y **exige** `ADMINPASSWORD`: si falta, el
entrypoint falla porque el server pide la clave por consola.

```bash
# en el VPS
sudo docker logs -f zomboid
```

Señales de arranque sano:

```
*** SERVER STARTED ***
*** Steam is enabled
Server is listening on port 16261 (for Steam connection) and port 16262 (for UDPRakNet connection)
RCON: listening on port 27015
```

El mundo vive en el volume Docker `cggpgaytrcmy8kimlw4tn45l_zomboid-data`, montado
en `/home/steam/Zomboid`. Los archivos de configuracion quedan en
`/home/steam/Zomboid/Server/<SERVERNAME>.ini` y el mapa en
`/home/steam/Zomboid/Saves/Multiplayer/<SERVERNAME>/`.

### ADMINPASSWORD ya no esta en el compose

El compose **no** declara `ADMINPASSWORD`: el entrypoint la pasa como argumento del
server y queda en claro en los logs. El mundo ya esta creado, asi que la cuenta
admin conserva su clave inicial (guardada en el archivo de secretos del backup).

Si hay que **recrear el mundo** (borrar el volume o cambiar `SERVERNAME`):

1. Descomentar `ADMINPASSWORD: ${ADMINPASSWORD}` en `docker-compose.yml`.
2. Setear la variable en Coolify (Resource > Environment Variables).
3. Deploy, esperar `*** SERVER STARTED ***`.
4. Borrar la variable de Coolify, volver a descomentarla en el compose y redeployar.

## Puertos

| Puerto | Protocolo | Para qué |
|---|---|---|
| 16261 | UDP | puerto principal del juego (`PORT`) — **obligatorio** |
| 16262 | UDP | `UDPPORT`, "direct connection port" — **obligatorio** |
| 8766 / 8767 | UDP | cliente de Steam (legacy en B42) |
| 16262-16272 | TCP | rango de slots (8) — se mantiene por requerimiento |
| 27015 | TCP | RCON (administración remota), solo LAN/Tailscale |

Con 8 slots el rango TCP es `16262-16272`. Si se sube la cantidad de jugadores,
ampliar el rango (y su port forwarding) al mismo tiempo.

> **Importante (Build 42)**: el server abre el socket **UDP 16261**; el trafico de
> juego no va por TCP. Por eso un checker de puertos TCP puede marcar el server
> como "cerrado" aunque los jugadores entren bien. Lo que hay que abrir en el
> router es **UDP 16261 y UDP 16262**.

## Slots (`MaxPlayers`)

El entrypoint **no** expone `MaxPlayers`: el server arranca con 32 y hay que
cambiarlo a mano en el INI (el valor por defecto de PZ es 32, no 8).

```bash
# en el VPS (el INI vive en el volume, sobrevive a los redeploys)
VOL=/var/lib/docker/volumes/cggpgaytrcmy8kimlw4tn45l_zomboid-data/_data
sudo sed -i 's/^MaxPlayers=.*/MaxPlayers=8/' $VOL/Server/ZomboidMGTSolutions.ini
sudo docker restart $(sudo docker ps -q --filter name=zomboid)
```

Si se recrea el mundo, hay que volver a aplicarlo.

## Jugadores

- Entrar por **IP:puerto** (`181.164.89.26:16261` desde internet, `100.75.207.93:16261`
  desde Tailscale). Project Zomboid no usa dominios para conectarse.
- `NOSTEAM=True` (configurado) permite entrar **sin cuenta de Steam** por TCP.
  Con `NOSTEAM=False` el server solo acepta clientes de Steam (y deja de abrir
  el puerto TCP principal).
- `PUBLIC=true` muestra el server en el navegador del juego. Con `NOSTEAM=True`
  el server no se anuncia en el master de Steam, asi que la via normal es la IP.
- El server no tiene password de entrada (`Password=` vacio en el INI): cualquiera
  que conozca la IP puede entrar. Para cerrarlo, poner `NOSTEAM=False`.
- Cuenta admin: `ADMINUSERNAME` / `ADMINPASSWORD` (solo la primera vez; después se
  cambia desde adentro del juego).

## Mods

Variables `WORKSHOP_IDS` (Steam Workshop) y `MOD_IDS` (mods locales), separadas por `;`.
Ej: `WORKSHOP_IDS=2166000000;2166000001`.

- Los mods del Workshop **se descargan en el segundo arranque** (el primero crea el mundo).
- Con la lista vacía el entrypoint **borra** `Mods`/`WorkshopItems` del INI.
- Para administrar la lista a mano: `SELF_MANAGED_MODS=1` y se edita el INI del server
  (`/home/steam/Zomboid/Server/<SERVERNAME>.ini`).
- Cada mod agrega tiempo de arranque y RAM: con 8 slots y muchos mods, subir `MEMORY`.

## Configuración del mundo

El archivo de reglas del server es
`/home/steam/Zomboid/Server/<SERVERNAME>.ini` (dentro del storage de Coolify).
Editarlo a mano (vía SSH) y reiniciar el contenedor para que aplique.

Ojo: el entrypoint reescribe algunos valores en cada arranque (`Public`, `PublicName`,
`UDPPort`, `Password`, `RCONPassword`, `Mods`, `WorkshopItems`, `Map`). Lo que no
esté en esa lista se conserva.

## Reinicio y guardado

El entrypoint maneja `SIGTERM`: manda `quit` a la consola del server y espera a que
el mundo se guarde. Por eso `stop_grace_period: 120s` en el compose — **no lo bajes**,
con los 10s por defecto de Docker el server muere a mitad de guardado.

Reiniciar desde Coolify (Restart) o:

```bash
sudo docker restart zomboid
```

## Backups

Lo unico que hay que respaldar es el volume del mundo
(`cggpgaytrcmy8kimlw4tn45l_zomboid-data` -> `/home/steam/Zomboid`): mapa, saves,
INIs de configuracion y logs. Copiar en caliente puede dar un mundo inconsistente:
detener el contenedor, copiar, levantar.

```bash
# en el VPS
C=$(sudo docker ps -q --filter name=zomboid)
VOL=/var/lib/docker/volumes/cggpgaytrcmy8kimlw4tn45l_zomboid-data/_data

sudo docker stop -t 120 $C
sudo tar czf /data/zomboid-backup-$(date +%F).tar.gz -C $VOL .
sudo docker start $C
```

Para restaurar: detener el contenedor, descomprimir sobre `$VOL` y volver a levantar.

En Coolify no hay S3 configurado, asi que el destino del backup hay que elegirlo
(rsync a otra maquina, S3 externo, etc.). El volume `..._zomboid-workshop` solo
guarda los mods descargados y se puede re-descargar.

## Diagnostico

```bash
sudo docker ps --filter name=zomboid          # estado
sudo docker logs --tail 100 $(sudo docker ps -q --filter name=zomboid)   # logs
sudo ss -ulnp | grep 16261                    # puerto escuchando
sudo docker port $(sudo docker ps -q --filter name=zomboid)   # puertos publicados
sudo docker inspect $(sudo docker ps -q --filter name=zomboid) --format '{{json .Mounts}}'
```

Si el server no aparece en el navegador del juego: `PUBLIC=true`, puerto 16261/UDP
abierto en el router, y `STEAMVAC=True` para servidores de Steam.

## Memoria y CPU

El VPS tiene 4 vCPU y 8 GB de RAM (compartidos con Coolify e InsForge).

| Ajuste | Valor | Donde |
|---|---|---|
| Heap de la JVM | `MEMORY=3072m` | variable de entorno en Coolify |
| Limite de RAM del contenedor | `mem_limit: 5g` | `docker-compose.yml` |
| Limite de CPU del contenedor | `cpus: 3` | `docker-compose.yml` |

**Los limites del panel (Resource > Memory/CPU) NO se aplican a apps Docker
Compose**: Coolify solo los usa con apps que tienen Dockerfile. Si se cambian los
limites, editar el compose y redeployear.

Subir `MEMORY` por encima de `3072m` solo si el host no reporta presion de memoria
(`free -m`): con el VPS compartido, un heap de 4 GB deja al host sin margen.

