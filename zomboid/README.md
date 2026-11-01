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
| 16262 | UDP | `UDPPORT`, "direct connection port" (RakNet) — **obligatorio** |
| 16262-16272 | TCP | rango de slots (8) — se mantiene por requerimiento |
| 27015 | TCP | RCON (administración remota), solo LAN/Tailscale |

En Build 42 **no** hacen falta los puertos 8766/8767 del Steam GameServer: el
server usa SteamNetworkingSockets (SDR), sale hacia Valve por puertos efímeros y
Steam enruta a los clientes. Por eso tampoco se publican `STEAMPORT1/2` (y B42
los rechaza con `unknown option "-steamport1"` en los logs).

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

Dos formas de entrar:

- **Listado público del juego**: pestaña *Internet* del menú multijugador. El
  server aparece como `[ARG] Piji Zomboid` (8 slots, sin password).
- **IP directa**: `181.164.89.26:16261` (internet), `192.168.68.222:16261` (LAN),
  `100.75.207.93:16261` (Tailscale). Project Zomboid no usa dominios.

`NOSTEAM=False` es **obligatorio** para figurar en el listado: el navegador de
servidores del juego se alimenta del master server de Steam
(`zombie/core/znet/ServerBrowser` solo llama a `n_RefreshInternetServers`), y con
`-nosteam` el server nunca se registra. `PUBLIC=true` en el INI mantiene el
anuncio activo.

El server no tiene password de entrada (`Password=` vacío en el INI): cualquiera
que lo vea en el listado puede entrar.

- Cuenta admin: `ADMINUSERNAME` / `ADMINPASSWORD` (solo la primera vez; después se
  cambia desde adentro del juego).

## Listado público (Steam)

Con `NOSTEAM=False` el server abre `SteamAPI_Init()`, se registra en Steam y
Steam lo publica con la IP/puerto reales. En los logs se ve:

```
Loading steam_api...
SteamUtils initialised successfully
Waiting for response from Steam servers
*** Steam is enabled
Server is listening on port 16261 (for Steam connection) and port 16262 (for UDPRakNet connection)
```

El `[S_API FAIL] Tried to access Steam interface SteamNetworkingUtils004 before
SteamAPI_Init succeeded` es una carrera del arranque y no impide el registro.

Verificación sin abrir el juego (Steam Web API, pública):

```bash
curl -s "https://api.steampowered.com/ISteamApps/GetServersAtAddress/v1/?addr=181.164.89.26:16261"
```

Tiene que devolver `"name":"[ARG] Piji Zomboid"` con `"max":8`, `"password":0` y
`"appid":108600`. Si devuelve `"servers":[]`, revisar `NOSTEAM`, `PUBLIC` y que el
contenedor haya terminado de cargar el mundo.

La IP que se anuncia NO se deduce sola (el contenedor ve `10.0.3.3`): va en el INI
del server, y sobrevive a los redeploys porque el mundo es persistente.

```bash
# con el contenedor DETENIDO (si no, el server reescribe el INI al guardar)
VOL=/var/lib/docker/volumes/cggpgaytrcmy8kimlw4tn45l_zomboid-data/_data
sudo sed -i 's|^server_browser_announced_ip=.*|server_browser_announced_ip=181.164.89.26|' \
  $VOL/Server/ZomboidMGTSolutions.ini
```

Si se cambia la IP pública del ISP hay que repetir este paso.

### Anuncio a projectzomboid.com

`zombie/util/PublicServerUtil` hace un GET a
`https://www.projectzomboid.com/server_browser/serverVar.php` sin fijar
User-Agent. Java manda `Java/17.x` y Cloudflare responde **403** (managed
challenge). Por eso el compose define:

```yaml
JAVA_TOOL_OPTIONS: '-Dhttp.agent="Mozilla/5.0 ... Chrome/141.0.0.0 Safari/537.36"'
```

Las comillas internas son obligatorias: `JAVA_TOOL_OPTIONS` se parte por espacios
y sin ellas el JVM no arranca (`Unrecognized option: (Windows`), lo que además
rompe el chequeo de `start-server.sh` y deja el contenedor en crash-loop con
`Only 64bit is supported`.

Ese endpoint ya no alimenta ninguna lista pública útil (el panel
`/server_browser/new/public/` está vacío para todos), así que esto es solo
higiene: el listado que importa es el de Steam.


## Mods

El set actual es un espejo del cliente: **155 mods / 125 items del Workshop**
(~2.5 GB, casi todos vehículos KI5 + `damnlib` + libs de ropa/UI). No hay mods de
mapa, así que `Map=` sigue en `Muldraugh, KY`.

Se administra con **`SELF_MANAGED_MODS=True`** y las listas escritas directo en el
INI, no con `WORKSHOP_IDS`/`MOD_IDS`:

```ini
Mods=<ids separados por ;>
WorkshopItems=<ids numericos del Workshop separados por ;>
```

Motivo: `MOD_IDS` son ~2.1 KB y hay ids con apóstrofes y espacios
(`GanydeBielovzki's Frockin Backstage!`). Pasar eso por una variable de entorno
depende de cómo la escape Coolify en el `.env` del compose, y si lo hace con
comillas simples la variable se corta. En el INI no hay quoting: es el formato
que el juego lee.

Con `SELF_MANAGED_MODS=True` el entrypoint **no toca** `Mods`/`WorkshopItems`;
sin él, cada arranque reescribe esas líneas desde el entorno (y con el entorno
vacío las borra).

### Replicar la lista desde el cliente

1. En el cliente, los mods que carga el juego quedan en
   `%USERPROFILE%\Zomboid\Logs\<fecha>_DebugLog.txt` como líneas
   `LOG : Mod f:0> loading <modid>.`
2. El id del Workshop de cada mod sale de
   `F:\Steam\steamapps\workshop\content\108600\<workshopid>\**\mod.info` (campo `id=`).
3. `WorkshopItems` = los 125 ids numéricos; `Mods` = los 155 ids de mod.

### Operación

- El server **descarga los items del Workshop al arrancar** (Steam los baja a
  `steamapps/workshop/content/108600`, volumen `..._zomboid-workshop`). La primera
  vez tardó ~20 min (2.5 GB de descarga + carga). Con los items ya en el volumen el
  arranque completo es de **~70 s**.
- **Linux es case-sensitive y el Workshop no.** Varios mods referencian carpetas en
  minúsculas (`media/animsets/...`, `mods/bandits/...`) que existen como
  `AnimSets`/`Bandits`. En Windows (cliente) resuelve igual; en el server no, y se
  pierden esos anim sets. Fix aplicado en el volumen (no está en el repo):
  22 symlinks `animsets -> AnimSets` en cada `media/` y uno
  `mods/bandits -> Bandits` en el item `3268487204`. Si una actualización del
  Workshop borra los symlinks, vuelven a aparecer `FileNotFoundException` en los
  logs (no impide arrancar).
- Con este set: `MEMORY=4096m` en Coolify y `mem_limit: 6g` en el compose. Con
  `3072m`/`5g` el cgroup mataba el contenedor.
- Si aparece `mod mismatch` al conectar, el cliente tiene una versión distinta de
  algún mod: actualizar sus suscripciones en Steam.
- Si aparece **"La version del articulo del Workshop es diferente a la del servidor"**,
  el server tiene una copia vieja de algun item. Los versionados viven en
  `appworkshop_108600.acf` (`manifest` + `timeupdated` por item): ese archivo del
  volume `..._zomboid-workshop` tiene que coincidir con el de Steam del cliente
  (`F:\Steam\steamapps\workshop\appworkshop_108600.acf`). Se resuelve reiniciando el
  contenedor: al arrancar detecta `NeedsUpdate` y baja la version nueva.
- Un mod nuevo en el cliente **no** se instala solo: hay que agregar su id a
  `WorkshopItems` y su(s) mod id(s) a `Mods` en el INI (con el contenedor detenido).


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

Si el server no aparece en el navegador del juego: `NOSTEAM=False` y
`PUBLIC=true`, y el contenedor tiene que haber llegado a `*** SERVER STARTED ***`.
Comprobar con `GetServersAtAddress` (ver *Listado público*). El reenvío de
16261/UDP en el router sigue siendo necesario para la conexión directa.

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

