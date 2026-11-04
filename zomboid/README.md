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

El set actual es un **espejo exacto del cliente**: **55 mods / 137 items del
Workshop** (sincronizado el 2026-11-04). No hay mods de mapa, así que `Map=` sigue
en `Muldraugh, KY`.

Los 55 mods son los que el cliente **carga** (en su orden de carga, que es el orden
del `Mods=`); los 137 items son **todos** los que el cliente tiene descargados en
`F:\Steam\steamapps\workshop\content\108600` (incluye los que no se activan, como
los que son solo mapas o utilidades). La diferencia importa: el `Mods=` define qué
se carga, el `WorkshopItems=` qué se descarga.

> El `Mods=` **no** lleva los nombres de los items del Workshop sino los
> `id=` de cada `mod.info` (ej. `PlayersOnMapRedux`, `RepairAnyClothes`), separados
> por `;`. Los `WorkshopItems=` sí llevan los ids numéricos de 10 dígitos.

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

### Build 42 exige carpeta de versión en cada mod

Un mod en Build 42 se registra **solo** si tiene una carpeta de versión:

```
<workshop>/<item>/mods/<ModId>/
├── mod.info              # raíz (B41)
├── media/                # raíz (B41)
├── common/media/         # opcional, contenido común a todas las versiones
└── 42.20/                # ← obligatorio: la carpeta de versión
    ├── mod.info          #   id=..., versionMin=42.20.0
    └── media/            #   el contenido que se carga
```

Si el mod solo trae la raíz, el juego **no lo registra**: el arranque avisa

```
WARN : Mod ... at ZomboidFileSystem.loadModAndRequired> required mod "X" not found
```

y el mod nunca carga. Es lo que pasaba con **Players On Map** (item `2732804047`,
actualizado por última vez en 2024-10-20, layout B41 puro): estaba descargado,
`Ready` en Steam y en el `Mods=` del INI, pero sin carpeta de versión no cargaba
ni en el server ni en el cliente.

Se verifica mirando el arranque:

```bash
C=$(sudo docker ps -q --filter 'label=coolify.applicationId=8')
sudo docker logs --since "$(sudo docker inspect -f '{{.State.StartedAt}}' $C)" $C 2>&1 \
  | grep -E '> loading |required mod'
```

Cada línea `loading <modid>` es un mod registrado. Con el set actual:
**55 cargados / 55 declarados, 0 faltantes**.

El fix se aplica **sintetizando** la carpeta de versión a partir de la raíz
(`mod.info` de la raíz + `versionMin=`, y copia de `media/`), tanto en el server
como en el cliente, para que los dos queden idénticos. Lo hace
`zomboid_control.ensure_b42_layout()` (allowlist explícita por mod) y el
equivalente en PowerShell del cliente. Es idempotente: si la carpeta ya existe
con `mod.info` y `media/`, no toca nada.

### Carpetas de animaciones: `AdvancedAnimator` las exige

> Antes (2026-11-03) esto se dejaba pasar: los `NoSuchFileException` de
> `.../media/AnimSets` y `.../media/actiongroups` "eran normales" porque el juego
> prueba esas rutas en todos los mods. **Ya no lo son**: con 55 mods, el arranque
> tiraba **990** `NoSuchFileException` (`AdvancedAnimator$1.visitFileFailed`) y el
> cliente se quejaba al conectar.

AdvancedAnimator recorre, en cada arranque, para cada mod del Workshop:

```
<mod>/common/media/{AnimSets,actiongroups}
<mod>/<ver>/media/{AnimSets,actiongroups}
<mod>/<ver>/common/media/{AnimSets,actiongroups}
```

Si alguna no existe, tira el error. El fix es **crearlas vacías** (el juego solo
las recorre) — es la misma reparación que se aplicó al cliente para que los dos
queden idénticos. Lo hace `zomboid_control.ensure_anim_layout()` (idempotente) y
el watcher lo re-aplica después de cada re-descarga de Steam, porque al
re-bajar un item Steam borra esas carpetas.

```bash
sudo python3 /opt/backups/zomboid_control.py animlayout   # crea las que falten
```

Verificado 2026-11-04: **1588 + 14 carpetas creadas** y el arranque quedó con
**0 errores de `AdvancedAnimator`**.

### Replicar la lista desde el cliente

1. En el cliente, los mods que carga el juego quedan en
   `%USERPROFILE%\Zomboid\Logs\<fecha>_DebugLog.txt` como líneas
   `LOG : Mod f:0> loading <modid>.`
2. El id del Workshop de cada mod sale de
   `F:\Steam\steamapps\workshop\content\108600\<workshopid>\**\mod.info` (campo `id=`).
3. `WorkshopItems` = los 137 ids numéricos; `Mods` = los 55 ids de mod.

El script `InsForge\tools\zomboid-client-sync.ps1` automatiza el paso 3:
aplica el layout B42 en el cliente, sube el `appworkshop_108600.acf` del cliente
al VPS y muestra el diff server vs cliente.

### Sincronizar el server con el cliente (procedimiento manual)

Hacer que el server sea **idéntico** al cliente es un procedimiento de varios
pasos, porque "borrar del server lo que el cliente ya no tiene" no alcanza: el
`Map=` y el `search_folder.sh` del entrypoint **re-agregan** los mapas que
aparecen en el workshop.

Pasos (con **0 jugadores**; el server se detiene y se vuelve a encender):

1. **Exportar la referencia del cliente**:
   - `Mods=` → los `id=` de los `mod.info` de los mods que el cliente **carga**,
     en el orden de carga (sale de las líneas `loading <modid>` del `console.txt`).
   - `WorkshopItems=` → los ids numéricos de las carpetas descargadas en
     `F:\Steam\steamapps\workshop\content\108600`.
2. **Borrar del server los items que el cliente ya no tiene** — del **workshop
   (carpeta + ACF)**, no solo del INI. Si queda el item en el workshop, el
   `search_folder.sh` del entrypoint vuelve a copiar su mapa a `media/maps` y el
   `Map=` se re-mergea: es lo que pasó con **Fort Rock Ridge** (`2806456685`),
   cuyos chunks daban `invalid chunk` al cliente.
   Hay que borrarlo de tres lugares: `content/108600/<id>/`, la entrada del
   `appworkshop_108600.acf` y `media/maps/<nombre del mapa>/` dentro del contenedor.
3. **Escribir `Mods=`, `WorkshopItems=` y `Map=Muldraugh, KY`** en el INI.
4. **Regenerar el `WorldDictionary`**: borrar (con el server **detenido**)
   `WorldDictionary.bin`, `WorldDictionaryLog.lua` y `WorldDictionaryReadable.lua`
   de `Saves/Multiplayer/<SERVERNAME>/`. El server los regenera al arrancar.
   Con el diccionario viejo el cliente fallaba al conectar con
   `Missing dictionary script on client: Base.SMB_steps_FirePole` (entradas stale
   de un mod que el server ya no cargaba).
5. **Reiniciar** y verificar con `docker logs --since` (ver abajo).

Verificado el 2026-11-04: 55/55 mods cargados idénticos al cliente, 0
diferencias, 0 `required mod not found`, 0 `invalid chunk`, 0 `AdvancedAnimator`,
0 errores de `WorldDictionary`, `SMB_steps` = 0.

#### Verificar el arranque (importante)

El `DebugLog-server.txt` **acumula varios arranques en el mismo archivo** (el
nombre del archivo es del arranque previo), así que contarlo sobre el archivo da
números de arranques viejos. Para contar solo el arranque actual hay que usar el
stdout del contenedor:

```bash
C=$(sudo docker ps -q --filter 'label=coolify.applicationId=8')
sudo docker logs --since "$(sudo docker inspect -f '{{.State.StartedAt}}' $C)" $C 2>&1 \
  | grep -E 'required mod|invalid chunk|AdvancedAnimator|SMB_steps|Fort Rock|SERVER STARTED'
```

Los `ERROR`/`WARN` que quedan son ruido conocido del build de Steam (presente
también en el cliente): `AngelCodeFont failed to load page 0 texture
mainfont_0.png`, `XuiSkin Could not find icon`, `IsoPropertyType.lookupOrDefaultStr`,
`InventoryItem.loadItem data length not matching`, `Missing ThumpSound for
breakable object`.

### Auto-sync con Steam (watcher)

En el VPS corre `zomboid-mods-watch.timer` (cada 15 min, `OnBootSec=3min`,
`Persistent=true`):

1. Compara el `timeupdated` del ACF del server contra la Web API de Steam
   (`GetPublishedFileDetails`) y avisa qué items tienen versión nueva.
2. Si hay **0 jugadores** (consulta RCON): reinicia el contenedor para que el
   juego baje las versiones y avisa por Telegram.
3. Si hay **jugadores**: difiere (no corta la partida) y avisa una sola vez.
4. Recrea los symlinks case-insensitive que Steam borra al re-descargar un item.
5. Re-sintetiza el layout B42 de los mods de la allowlist.
6. Recrea las carpetas de animaciones (`ensure_anim_layout()`) que Steam también
   borra al re-descargar un item.
7. Re-aplica el estado deseado de los toggles de mapa (`restore_map_toggles()`),
   pero **solo con 0 jugadores** (los toggles 2 y 3 necesitan el server detenido).

```bash
sudo systemctl status zomboid-mods-watch.timer
sudo journalctl -u zomboid-mods-watch -f
sudo python3 /opt/backups/zomboid-mods-watch.py    # corrida manual
```

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
  logs (no impide arrancar). El watcher los recrea solo.
- Con este set: `MEMORY=4096m` en Coolify y `mem_limit: 6g` en el compose. Con
  `3072m`/`5g` el cgroup mataba el contenedor.
- Si aparece `mod mismatch` al conectar, el cliente tiene una versión distinta de
  algún mod: actualizar sus suscripciones en Steam.
- Si aparece **"La version del articulo del Workshop es diferente a la del servidor"**,
  hay un item del Workshop con versiones distintas entre cliente y server. Los
  versionados viven en `appworkshop_108600.acf` (`manifest` + `timeupdated` + `size`
  por item): ese archivo del volume `..._zomboid-workshop` tiene que coincidir con
  el de Steam del cliente (`F:\Steam\steamapps\workshop\appworkshop_108600.acf`).
  En el log del cliente el item culpable es el unico con
  `item state CheckItemState -> Fail ID=<id>` (los demas dan `Ready`).
  Procedimiento cuando un autor sube una version nueva:
  1. **Cerrar el juego cliente por completo** (no alcanza con reconectar: la
     instancia cachea el estado de los items al arrancar) y volver a abrirlo, para
     que Steam baje la version nueva.
  2. **Reiniciar el contenedor** (`docker restart`): el server solo consulta Steam
     al arrancar, baja lo que este desactualizado y announce la version actual.
  3. Verificar que ambos ACF coinciden antes de que entre alguien.
- Caso real (2026-11-01): el item `3826426915` (ProjectRPGExpanded / "Survivor PDA")
  fue resubido a las 13:40. El cliente lo bajo a las 13:42 pero con el juego abierto
  desde las 13:20, y el server lo actualizo a las 13:48. Con ambos lados en
  `timeupdated=1793551211` / `size=93304633` entra bien.
- Un mod nuevo en el cliente **no** se instala solo: hay que agregar su id a
  `WorkshopItems` y su(s) mod id(s) a `Mods` en el INI (con el contenedor detenido).
- Ojo al contar items del ACF: Steam lista **cada item dos veces**
  (`WorkshopItemsInstalled` + `WorkshopItemDetails`). Hay que contar ids únicos.

### Control por Telegram

El bot (`insforge-backup-bot`) tiene el menú `/pz`:

| Botón / comando | Qué hace |
|---|---|
| 📊 Estado / `/pz estado` | Encendido, jugadores, RAM, mods, diff con el cliente |
| 🔍 Buscar updates / `/pz updates` | Items con versión nueva en Steam |
| ⬇️ Aplicar todas / `/pz aplicar` | Reinicia para aplicar las updates pendientes |
| ♻️ Reiniciar / `/pz reiniciar` | Reinicio rápido del contenedor |
| ☁️ Redeploy / `/pz redeploy` | Redeploy real de Coolify (`POST /applications/{uuid}/restart`) |
| 📜 Log / `/pz log` | Últimas líneas del server |
| 🛑 Apagar / `/pz apagar` | Apaga el contenedor |
| ▶️ Encender / `/pz encender` | Enciende el contenedor |
| 👥 Usuarios / `/pz usuarios` | Listado de usuarios del server y sus roles |
| 🗺️ Mapa / `/pz mapa` | Submenú de los 3 toggles de mapa (ver abajo) |

Las acciones que reinician o apagan piden confirmación con botones (`Sí, hacelo` /
`Cancelar`), que expira a los 5 minutos. El **redeploy real** es `POST {COOLIFY_API}/applications/{uuid}/restart`
(no existe `/deploy`, da 404).

> **Cloudflare bloquea al VPS**: `https://coolify.mgtsolutions.uk/api/v1/*` desde el
> VPS devuelve **403 "Attention Required"** (bloquea la IP del datacenter; desde la
> PC local responde 200). Coolify corre en ese mismo host y escucha en
> `127.0.0.1:8000`, así que la librería usa `http://127.0.0.1:8000/api/v1` como
> fallback. Verificado: el redeploy recrea el contenedor y el server vuelve con
> `SERVER STARTED` y los 55 mods cargados.

### Usuarios y roles del server

Los usuarios no viven en el INI: están en la SQLite del volume
`.../db/<SERVERNAME>.db`, tabla `whitelist`, con un `role` que apunta a la tabla
`role` (roles jerárquicos de Build 42, con capabilities granulares):

| rol | Qué puede |
|---|---|
| `banned` | No puede entrar |
| `user` | Solo jugar (default al conectarse la primera vez) |
| `priority` | Prioridad de conexión, no lo kickean por lag |
| `observer` | Teleport, god mode, abrir puertas, noclip |
| `gm` | Game master: items, XP, cheats |
| `moderator` | Casi todo (kick, ban, server messages), sin editar opciones del server |
| `admin` | Todos los permisos. La cuenta `admin` de consola es el superusuario interno |

Gestión desde el bot (todo con efecto inmediato vía RCON, sin reiniciar):

| Comando | Qué hace |
|---|---|
| `/pz usuarios` | Lista usuarios con rol + último login (UTC-3) + Steam ID |
| `/pz alta <usuario> [password]` | Alta en el whitelist (queda `user`); sin password genera una aleatoria y la muestra |
| `/pz borrar <usuario>` | Lo saca del whitelist |
| `/pz rol <usuario> <rol>` | Cambia el rol (`setaccesslevel`); si el nuevo rol es `banned` lo expulsa |

Todas las operaciones (alta/borrar/rol) piden confirmación con botones y expiran
a los 5 minutos. La cuenta `admin` de consola está protegida: no se puede borrar
ni degradar desde el bot.

Comandos equivalentes del juego por RCON (los usa la librería):
`adduser <u> <pw>`, `removeuserfromwhitelist <u>`, `setaccesslevel <u> <rol>` —
los 7 roles se aceptan tal cual (`banned/user/priority/observer/gm/moderator/admin`)
y el server escribe la SQLite al instante.

La librería compartida está en
`/opt/backups/zomboid_control.py` y se puede usar desde la línea de comandos:

```bash
sudo python3 /opt/backups/zomboid_control.py status|pending|mods|players
sudo python3 /opt/backups/zomboid_control.py errors       # rutas y mods no cargados
sudo python3 /opt/backups/zomboid_control.py diff         # server vs cliente
sudo python3 /opt/backups/zomboid_control.py fixlinks     # recrea symlinks
sudo python3 /opt/backups/zomboid_control.py b42layout    # (re)sintetiza layout B42
sudo python3 /opt/backups/zomboid_control.py animlayout   # crea carpetas de animaciones
sudo python3 /opt/backups/zomboid_control.py users        # listado de usuarios
sudo python3 /opt/backups/zomboid_control.py useradd <u> [pw] | userdel <u> | userrole <u> <rol>
sudo python3 /opt/backups/zomboid_control.py map          # estado de los 3 toggles de mapa
sudo python3 /opt/backups/zomboid_control.py mapvis <1-3> | mapmod <on|off> | mapworld <on|off>
```


### Toggles de mapa

Hay **tres toggles distintos** de "mostrar/ocultar el mapa", cada uno con su
mecanismo. El bot los maneja desde `🗺️ Mapa`; el toggle 1 es en vivo, los 2 y 3
reinician el server (con confirmación en el bot):

| # | Toggle | Mecanismo | Efecto |
|---|---|---|---|
| 1 | 👁️ Ver jugadores en el mapa (vanilla) | Opción del server `MapRemotePlayerVisibility` | **En vivo**, sin reiniciar |
| 2 | 🧟 Overlay de jugadores en el mapa | SandboxVar `PlayersOnMapRedux.Enabled` en `Server/<SERVERNAME>_SandboxVars.lua` | `stop → edit → start` |
| 3 | 🗺️ Acceso al mapa del mundo | SandboxVar `AllowWorldMap` en `Server/<SERVERNAME>_SandboxVars.lua` | `stop → edit → start` |

Detalles de cada uno:

1. **`MapRemotePlayerVisibility`**: `1=ocultos`, `2=solo amigos`, `3=todos`.
   Aplica al instante con `changeoption` por RCON (verificado). El bot lo alterna
   con un botón (`ocultar todos` ↔ `ocultos`); CLI: `mapvis <1-3>`.
2. **Overlay `PlayersOnMapRedux`**: el overlay que ve el cliente lo aporta el mod
   **PlayersOnMapRedux** (Workshop item `3792135936`). Su única config en el
   server es la SandboxVar `SandboxVars.PlayersOnMapRedux.Enabled` — **no** un
   `Config.lua` del mod. CLI: `mapmod on|off` (hace el `stop → edit → start` solo).

   > ⚠️ El mod **PlayersOnMap** (item `2732804047`) **no** es el que usa el cliente
   > (el cliente carga `PlayersOnMapRedux`) y su `common/Config.lua` ni siquiera
   > existe en el server: no sirve para togglear el overlay. Es el único item
   > con el layout B41 parcheado (ver *Build 42 exige carpeta de versión*).

   El server **reescribe** el `.lua` de SandboxVars al apagarse, así que la
   edición tiene que hacerse con el contenedor **detenido**. Por eso el CLI y el
   bot orquestan `stop → edit → start` (no alcanza con editar y reiniciar).
3. **`AllowWorldMap`** (SandboxVars): misma mecánica — el server reescribe el
   `.lua` al apagarse, así que se edita con el contenedor **detenido**. CLI:
   `mapworld on|off`. No toca `AllowMiniMap`.

El estado deseado de los toggles 2 y 3 queda guardado en
`/opt/backups/zomboid-map-state.json` y el watcher (`zomboid-mods-watch`) lo
re-aplica con `restore_map_toggles()` si Steam re-descarga el item o el server
reescribe el `.lua` — **solo si hay 0 jugadores** (si no, difiere y avisa).

```bash
sudo python3 /opt/backups/zomboid_control.py map            # estado de los 3
sudo python3 /opt/backups/zomboid_control.py mapvis 3       # en vivo, sin reinicio
sudo python3 /opt/backups/zomboid_control.py mapmod off     # reinicia (stop→edit→start)
sudo python3 /opt/backups/zomboid_control.py mapworld on    # reinicia (stop→edit→start)
```

Verificado end-to-end en el VPS (2026-11-04, con 0 jugadores y 4 reinicios
reales): los tres toggles cambian el valor correcto, el server vuelve a anunciar
`SERVER STARTED` después de cada `stop → edit → start`, y el estado queda
restaurado. Test reproducible: `InsForge\tools\ops-map-test2.sh` (**19 PASS /
0 FAIL**).


## Configuración del mundo

El archivo de reglas del server es
`/home/steam/Zomboid/Server/<SERVERNAME>.ini` (dentro del storage de Coolify).
Editarlo a mano (vía SSH) y reiniciar el contenedor para que aplique.

Ojo: el entrypoint reescribe algunos valores en cada arranque (`Public`, `PublicName`,
`UDPPort`, `Password`, `RCONPassword`). Con `SELF_MANAGED_MODS=True` **no** toca
`Mods` ni `WorkshopItems`, pero **sí re-mergea `Map=`** al final y el
`search_folder.sh` copia `mods/<mod>/media/maps/*` a `media/maps` (si no existe).
Por eso un item de mapa que quede en el workshop **vuelve** al `Map=` por más
que lo borres del INI: hay que borrarlo del workshop (ver *Sincronizar el server
con el cliente*). Lo que no esté en esa lista se conserva.

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
| Heap de la JVM | `MEMORY=4096m` | variable de entorno en Coolify |
| Limite de RAM del contenedor | `mem_limit: 6g` | `docker-compose.yml` |
| Limite de CPU del contenedor | `cpus: 3` | `docker-compose.yml` |

**Los limites del panel (Resource > Memory/CPU) NO se aplican a apps Docker
Compose**: Coolify solo los usa con apps que tienen Dockerfile. Si se cambian los
limites, editar el compose y redeployear.

Subir `MEMORY` por encima de `4096m` solo si el host no reporta presion de memoria
(`free -m`): con el VPS compartido, un heap de 4 GB deja al host sin margen.
Con `3072m`/`5g` el cgroup mataba el contenedor (el server llega a ~4.9 GiB con
los 55 mods).

