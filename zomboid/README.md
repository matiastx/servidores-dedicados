# Server de Project Zomboid

Runbook operativo del server dedicado. Imagen:
[`danixu86/project-zomboid-dedicated-server`](https://github.com/Danixu/project-zomboid-server-docker)
(tag `42.21-release-2`).

> **Estado**: recreado desde cero el **2026-11-04**. Mundo nuevo, INI nuevo,
> SandboxVars nuevas y DB de usuarios nueva. Se conservaron los items de
> Workshop (hoy **138**, identicos a los del cliente) y se restauró la whitelist.
>
> **2026-11-05**: `SELF_MANAGED_MODS=True` (el INI manda, lo gestiona el bot) y
> nuevo boton **🧩 MODS = Cliente** que deja el server identico al cliente.

## Resumen del estado actual

| Que | Valor | Donde vive |
|---|---|---|
| Mods cargados | **56** (los mismos que carga el cliente) | `Mods=` en el INI |
| Items de Workshop | **138** | `WorkshopItems=` en el INI |
| Referencia del cliente | union 56 mods / 138 items | `/opt/backups/zomboid-client-*` |
| Preset de partida | `Apocalypse` | `SERVERPRESET` |
| Mapa | `Muldraugh, KY` | `Map=` en el INI |
| MaxPlayers | `8` | `MaxPlayers=` en el INI |
| Jugadores maximos | 8 slots (TCP `16262-16272`) | compose |
| RCON | `127.0.0.1:27015` (solo loopback) | compose |
| Mundo | nuevo (2026-11-04) | volume `zomboid-data` |
| Arranque | ~80 s con los 56 mods | - |

## Arranque

El contenedor lo levanta Coolify (o a mano, ver [Redeploy manual](#redeploy-manual)).

```bash
# en el VPS
sudo docker logs -f zomboid-cggpgaytrcmy8kimlw4tn45l-024011473819
```

Senales de arranque sano:

```
*** SERVER STARTED ***
*** Steam is enabled
Server is listening on port 16261 (for Steam connection) and port 16262 (for UDPRakNet connection)
RCON: listening on port 27015
```

Con los 56 mods el arranque tarda **~80 segundos**. Es normal: el mundo nuevo se
genera la primera vez.

El mundo vive en el volume Docker `cggpgaytrcmy8kimlw4tn45l_zomboid-data`, montado
en `/home/steam/Zomboid`. Los archivos de configuracion quedan en
`/home/steam/Zomboid/Server/<SERVERNAME>.ini` y el mapa en
`/home/steam/Zomboid/Saves/Multiplayer/<SERVERNAME>/`.

### ADMINPASSWORD: solo el primer arranque

El entrypoint **exige** `ADMINPASSWORD` la primera vez: crea la cuenta `admin`
y el mundo. Sin ella el arranque falla pidiendo la clave por consola.

El compose la declara (`ADMINPASSWORD: ${ADMINPASSWORD}`) pero la variable
**no esta definida** en el `.env` del VPS ni en Coolify: compose avisa
`"ADMINPASSWORD" variable is not set` (ruido inofensivo) y el server arranca
igual porque el mundo ya existe.

- El entrypoint pasa la clave como argumento del server, asi que **queda en
  claro en los logs** del contenedor. Por eso no se deja puesta en produccion.
- La clave real quedo **hasheada (bcrypt) en la DB del juego**
  (`zomboid-data/db/ZomboidMGTSolutions.db`), no en el `.env` ni en Coolify.
- Para **recrear el mundo**: setear `ADMINPASSWORD` en Coolify, arrancar **una
  vez** y volver a borrarla.

## Mods

### Como funcionan

La imagen **no descarga** el Workshop. El entrypoint solo escribe
`WorkshopItems=` y `Mods=` en el INI a partir de `WORKSHOP_IDS` / `MOD_IDS`, y
el juego verifica lo que hay en el volume `zomboid-workshop`.

Por eso:

- `zomboid-workshop` se conserva entre recreaciones (2,6 GB; borrarlo obliga a
  re-descargar los 137 items).
- `zomboid-data` (mundo + INI + SandboxVars + DB) si se borra, para empezar de
  cero. Ver [Recrear el mundo](#recrear-el-mundo-desde-cero).

### Formato de `MOD_IDS`: plano, sin backslashes

`Mods=RepairAnyClothes;89def110;FH;...` — **plano**, sin el prefijo `\` que
pide el README upstream de la imagen.

El prefijo `\` de la imagen es un marcador de *id de mod*, no un prefijo
obligatorio: `TombBodyCompat` declara `require=\TombBody` en su `mod.info` y el
cliente lo carga **igual** usando una lista plana. Verificado de las dos
formas:

- `set_ini_option` de la imagen escribe el valor **plano** en las dos ramas
  (append y replace) y queda estable entre reinicios.
- El server levanta **56/56 mods, 0 `required mod not found`** con la lista
  plana.

El orden de `MOD_IDS` es el del `Mods=` del server (derivado del cliente) con
`TombBodyCompat` insertado justo despues de `TombBody`. El juego usa ese
orden como orden de carga.

### Donde se editan

Desde el 2026-11-05 la verdad vive en **el INI del server**, no en el compose:

| Variable | Valor | Que hace |
|---|---|---|
| `SELF_MANAGED_MODS` | `True` | el entrypoint **no** toca `Mods=` / `WorkshopItems=`; quedan bajo control del bot |
| `MOD_IDS` | 56 ids, 894 chars | se mantiene **alineado** con el INI (el bot lo actualiza junto con el INI) |
| `WORKSHOP_IDS` | 138 ids, 1518 chars | idem |

> **Por que `True`.** Con `False` el entrypoint reescribe `Mods=` y
> `WorkshopItems=` en cada arranque a partir del env, asi que cualquier cambio
> escrito a mano en el INI se pierde al reiniciar y el bot depende de la API de
> Coolify para poder aplicar un cambio. Con `True` el INI manda y el bot puede
> aplicar los cambios reiniciando el contenedor (90 s) sin depender de la API.
> El bot sigue actualizando `MOD_IDS` / `WORKSHOP_IDS` en Coolify, asi que las
> dos fuentes nunca divergen y volver a `False` es seguro.

Las variables viven **cifradas en la DB de Coolify** (`environment_variables.value`
tiene cast `encrypted`), duplicadas por fila (buildtime + runtime). Se editan
desde el panel de Coolify o con la API (`PATCH /applications/{uuid}/envs`,
matchea por `key`); para tocar la DB a mano:

```bash
# dentro del contenedor de Coolify, con bootstrap de Laravel (NO tinker inline:
# el quoting de PowerShell rompe los heredocs)
php /tmp/coolify-update.php
```

### MODS = Cliente (botón del bot)

El botón **🧩 MODS = Cliente** del bot deja el server **idéntico** al cliente.
El cliente es la fuente de verdad y su referencia vive en el VPS, subida desde
la PC:

| Archivo en el VPS | Que es |
|---|---|
| `/opt/backups/zomboid-client-mods.txt` | **union** de los mods que carga el cliente, en orden de carga |
| `/opt/backups/zomboid-client-mods-f0.txt` | **capa 1**: mods locales del cliente (`f:0`) |
| `/opt/backups/zomboid-client-mods-st0.txt` | **capa 2**: mods que manda el server (`f:0 st:0`) |
| `/opt/backups/zomboid-client-ids.txt` | ids de Workshop descargados en el cliente |
| `/opt/backups/zomboid-client-ref.json` | metadata (fecha, conteos) |

La referencia se genera con `G:\Proyectos\InsForge\tools\ops-client-export.ps1`
(lee `C:\Users\matia\Zomboid\console.txt` y el workshop local, y sube todo al
VPS). **Hay que correrla despues de jugar/conectar**, porque el log del juego es
la unica fuente que dice que mods cargo el cliente y en que orden.

Que hace el bot al sincronizar:

1. Compara `Mods=` / `WorkshopItems=` del server contra la referencia y arma el
   plan (agregar/quitar mods, agregar/quitar items, ajustar el orden).
2. Pide **confirmacion** (expira a los 5 min) y avisa si hay jugadores.
3. Detiene el server, escribe `Mods=` y `WorkshopItems=` en el INI y borra del
   workshop las carpetas que no tiene el cliente.
4. **Conserva el orden actual** del server para los mods que ya tiene y
   **inserta los mods nuevos** en la posicion que les da el cliente. Si el
   cliente impose otro orden, se adopta el del cliente.
5. Recrea los symlinks case-insensitive y las carpetas `AnimSets` /
   `actiongroups` (AdvancedAnimator las exige).
6. Actualiza `MOD_IDS` / `WORKSHOP_IDS` en Coolify para que no divergan.
7. Encende el server y **espera a que el juego baje los items nuevos** del
   Workshop; recien ahi repara el layout de esos items.
8. Verifica que el server cargo los mismos mods que el cliente. Si falta alguno,
   reinicia una vez mas.

El bot va reportando el **avance en %** en el mismo mensaje de Telegram
(barra de progreso, ~1 actualizacion cada 4 s).

> El bot **no** copia el ACF del cliente al server: ese archivo es el estado
> de Steam del workshop y copiarlo haria que el server creyera que ya tiene
> los items descargados.

Verificado el 2026-11-05 con el caso real: el cliente tenia **138** items de
Workshop y el server **137** (faltaba `3832274571` = FloorAttackHighlightFix).
El bot agrego el item, espero la descarga, reparo el layout y quedo
`IDENTICO: True` con 56/56 mods cargados.

### Los 56 mods

Los 56 que carga el cliente (orden de carga):

```
Expanded_Belt_B42;FixedLightOnBeltAF;NeatUI_Framework;WhoseKeyIsThis;FittedBarricades;
CyesPushDoors;FittedSheets;ZombieVisibilityFix;DetailedPlayerTextureFemaleB42;
TombBody;TombBodyCompat;SPNCC;TombBodyCustom;TombBodyTex;Bandits2;SpnOpenClothBase;
SpnOpenCloth;SpnHairAPI;FH;SpnHair;SpnCloth;SPNCCDetails;SPNCCDetailsHD;
RainCleansBlood;P4HasBeenRead;FemaleModelFix;EQUIPMENT_UI;CleanUI;RepairAnyClothes;
89def110;87cruiser;EasyConfigChucked;GenRange;ChuckleberryFinnAlertSystem;
SPNRetexture;SPNRetextureCustom;SPNRetextureUnderwear;SPNRetextureZombie;
SPNRetextureZombieUnderwear;Project_Seasons_B42;Project_Seasons_B42_LITE;
Project_Seasons_B42_NORUST;Project_Seasons_B41;Project_Seasons_B41_LITE;SPNCCFaces;
TombBodyTexDOLL;TombBodyTexNUDE;AmbientCityLights;DynamicHordesBCUltimateSurvival;
TrueSleep;VanillaOutfitsExpanded;CarWeaponFix;BetterCorpseBurning;WashAll;
PlayersOnMapRedux;TrueLineOfSight
```

> `TombBodyCompat` es el mod 56: estaba en la lista del cliente
> (`mods/default.txt`) y el cliente lo carga. La referencia anterior tenia 55 y
> se quedo desactualizada.

### El watcher de Workshop

`zomboid-mods-watch.timer` corre **cada 15 min** en el VPS. Por pasada:

1. Consulta la Web API de Steam y compara con el ACF del server.
2. Si hay items desactualizados: con **0 jugadores** reinicia el contenedor y
   avisa por Telegram; con jugadores avisa una vez y difiere (no corta la
   partida).
3. Recrea los **symlinks case-insensitive** que Steam borra al re-descargar.
4. Recrea las carpetas **`AnimSets` / `actiongroups`** que AdvancedAnimator exige.

Sobre el punto 4: **medido**. Sin esas carpetas el arranque tira **505 errores
de `AdvancedAnimator`**; con ellas, **0**. No es opcional.

> **Hack de layout B42 eliminado.** Antes el watcher sintetizaba una carpeta
> `<mod>/42.20/` para los mods que solo traian layout B41. Desde la recreacion
> todos los mods del cliente traen layout B42 nativo, asi que el server carga
> 56/56 sin hacks. Se limpio el residuo sintetico de `PlayersOnMap`.

## INI: ajustes manuales

El INI vive en `zomboid-data/Server/ZomboidMGTSolutions.ini`. El entrypoint lo
genera desde el preset, asi que **todo ajuste manual se pierde si se recrea el
mundo**.

Estos son los valores que hay que re-aplicar (el server llega con los del
preset, que son distintos):

| Clave | Valor a setear | Por que |
|---|---|---|
| `MaxPlayers` | `8` | el preset pone 32; el server tiene 8 slots (TCP `16262-16272`) |
| `server_browser_announced_ip` | `181.164.89.26` | sin esto el server no se anuncia en el listado |
| `AdminSafehouse` | `true` | el preset lo deja en `false` |
| `PlayerSafehouse` | `true` | idem |
| `AnnounceDeath` | `true` | idem |
| `HideAdminsInPlayerList` | `true` | idem |
| `HidePlayersBehindYou` | `false` | idem |
| `SleepAllowed` | `true` | idem |

`ResetID`, `Seed` y `ServerPlayerID` **no** se copian del mundo viejo: son la
identidad del mundo nuevo.

Para editar el INI el server tiene que estar **detenido** (el juego lo
reescribe al apagarse):

```bash
sudo docker compose -f /data/coolify/applications/<uuid>/docker-compose.yaml \
  --project-directory /data/coolify/applications/<uuid> stop
# ... editar ...
sudo docker compose ... up -d
```

## SandboxVars y toggles de mapa

El mod `PlayersOnMapRedux` se controla desde el bloque
`PlayersOnMapRedux` de `ZomboidMGTSolutions_SandboxVars.lua`:

```lua
PlayersOnMapRedux = {
    Enabled = true,
    ...
}
```

El preset `Apocalypse` deja `AllowWorldMap=true` y `AllowMiniMap=false`. El
bloque `PlayersOnMapRedux` lo agrega el mod, no el preset.

Los tres toggles del bot:

| Comando | Que cambia | Reinicia |
|---|---|---|
| `mapvis` | `MapRemotePlayerVisibility` (visibilidad vanilla en el mapa) | no, en vivo por RCON |
| `mapmod` | `PlayersOnMapRedux.Enabled` (overlay) | si (`stop -> edit -> start`) |
| `mapworld` | `AllowWorldMap` (acceso al mapa del mundo) | si (`stop -> edit -> start`) |

El estado deseado se guarda en `/opt/backups/zomboid-map-state.json` y
`restore_map_toggles()` lo re-aplica, **solo con 0 jugadores conectados**.

## Whitelist y usuarios

La DB del juego (`zomboid-data/db/ZomboidMGTSolutions.db`) tiene:

| Tabla | Contenido |
|---|---|
| `whitelist` | `admin` (rol 7), `Matiastx` (rol 7), `JoaKiga` (rol 2) |
| `role` | 7 roles |
| `capabilities` | 248 permisos |
| `defaultRoles` | 9 roles por defecto |
| `userlog` | historial de conexiones |

Restaurada desde el backup del mundo anterior: los hashes bcrypt de `Matiastx` y
`JoaKiga` son los originales (siguen entrando con su clave de siempre), y la
cuenta `admin` quedo con la clave nueva (hash distinto al del backup, verificado).

El bot gestiona la whitelist **por RCON** (`/pz alta`, `/pz borrar`, `/pz rol`),
que es seguro con el server corriendo. Para restaurar hashes a mano hay que
apagar el server (el juego tiene la SQLite abierta).

Con el bot todo se hace a botones: **Usuarios** -> usuario -> **🔑 Cambiar
rol** (un boton por rol) o **🗑️ Eliminar usuario**. El rol `banned` ademas
expulsa al jugador si esta conectado.

## RCON

El RCON escucha en `27015` y esta publicado **solo en loopback**:

```yaml
- "127.0.0.1:27015:27015/tcp"
```

> **Landmine corregido.** La columna `docker_compose` de la DB de Coolify
> (la que Coolify despliega) publicaba `27015:27015/tcp` -> bound a `0.0.0.0`,
> alcanzable desde la LAN y desde los otros contenedores de Coolify. Ahora las
> dos columnas de la DB (`docker_compose` y `docker_compose_raw`) tienen el
> loopback. Verificar antes de cada redeploy:

```bash
sudo docker exec coolify-db psql -U coolify -d coolify -tAc \
  "SELECT docker_compose FROM applications WHERE id=8" | grep 27015
```

El bot corre en el host y conecta a `127.0.0.1:27015`
(`zomboid_control.RCON_HOST`). **Nunca** hacer port forwarding del 27015.

## Coolify: app id 8

| Dato | Valor |
|---|---|
| Aplicacion | id `8` |
| UUID | `cggpgaytrcmy8kimlw4tn45l` |
| Compose en disco | `/data/coolify/applications/cggpgaytrcmy8kimlw4tn45l/docker-compose.yaml` |
| `.env` en disco | `/data/coolify/applications/cggpgaytrcmy8kimlw4tn45l/.env` |

Coolify mantiene **dos** copias del compose en la DB:

- `docker_compose`: la que despliega (tiene labels, volumes absolutos, red
  externa). Es la que hay que mantener en sync con el compose del repo.
- `docker_compose_raw`: la version "cruda" del editor de la UI. Si alguien
  redeploya desde el panel, esta es la que puede pisar el `docker_compose`.

### Redeploy manual

El bot tiene un boton **Redeploy** que usa la API de Coolify. El token se
regenero el 2026-11-05 y quedo funcionando (la API publica responde 403 de
Cloudflare desde el VPS, asi que el bot cae al endpoint local
`http://127.0.0.1:8000/api/v1`).

El token vive en `/opt/backups/coolify.env` (root-only) y **no** es el mismo que
el de `InsForge\.env.local` (ese da 401). Redeploy a mano:

```bash
cd /data/coolify/applications/cggpgaytrcmy8kimlw4tn45l
sudo docker compose up -d
```

## Bot de Telegram

`insforge-backup-bot.service`. Menu `/pz` (botones, una fila por fila):

| Boton | Que hace |
|---|---|
| **Estado** | estado del server |
| **Buscar updates** | items de Workshop desactualizados |
| **🧩 MODS = Cliente** | deja el server identico al cliente (ver [MODS = Cliente](#mods--cliente-boton-del-bot)) |
| **👥 Usuarios** | submenu con botones: listado -> usuario -> eliminar / cambiar rol |
| **Aplicar todas** | reinicia aplicando las updates de Steam |
| **Reiniciar** / **Apagar** / **Encender** | control del contenedor |
| **Redeploy** | redeploy via la API de Coolify |
| **Log** | ultimas lineas del server |
| **Mapa** | los tres toggles de la tabla de arriba |

El submenu de **Usuarios** es a botones: se muestra el listado (un boton por
usuario), al tocar uno aparecen **🔑 Cambiar rol** (un boton por rol) y
**🗑️ Eliminar usuario**, y al confirmar se ejecuta por RCON. Los comandos
`/pz alta`, `/pz borrar` y `/pz rol` siguen funcionando.

Las operaciones que cambian el estado (alta/baja/rol, sincronizacion de mods,
toggles que reinician) piden **confirmacion con boton**, y la confirmacion
**expira a los 5 minutos**
(`PZ_CONFIRMAR[accion] = time.time() + 300`).

El watcher y el bot avisan por Telegram cuando hay updates de Workshop y
cuando difieren por jugadores conectados.

## Reinicios: solo con 0 jugadores

Regla general del server: **nunca reiniciar ni recargar con gente jugando**.
Todos los scripts operativos hacen preflight de `players_connected()` y abortan
si no es 0. Si hay que reiniciar con jugadores, hace falta confirmacion
explicita del dueño.

## Recrear el mundo desde cero

Lo que se hizo el 2026-11-04, en orden:

1. **Preflight**: 0 jugadores, backups existentes, imagen presente.
2. **Backups** (ver [Backups](#backups)).
3. `docker compose down`.
4. `docker volume rm <uuid>_zomboid-data` — **solo el volume de datos**.
   El volume del workshop se conserva.
5. Escribir el `.env` nuevo (`MOD_IDS`, `WORKSHOP_IDS`,
   `SELF_MANAGED_MODS=False`, `ADMINPASSWORD` temporal).
6. `docker compose up -d` y esperar `SERVER STARTED` (~80 s).
7. **Restaurar la whitelist** con el server detenido (hashes bcrypt).
8. Quitar `ADMINPASSWORD` del `.env` y reiniciar para **comprobar que el mundo
   arranca sin ella**.
9. Re-aplicar los ajustes manuales del INI ([tabla](#ini-ajustes-manuales)).
10. Actualizar la DB de Coolify (env vars cifradas + `docker_compose`).

Los scripts de este paso viven en `G:\Proyectos\InsForge\tools\`
(`ops-deploy-recreate.sh`, `ops-restore-whitelist.sh`, `ops-final-check.sh`,
`ops-update-coolify-db.sh`). No se versionan: son de un solo uso.

### Verificacion del arranque

```bash
# contadores del arranque actual
sudo docker logs --since <StartedAt> <container> > /tmp/boot.log
grep -ac 'SERVER STARTED'       /tmp/boot.log   # 1
grep -ac 'required mod'          /tmp/boot.log   # 0
grep -ac 'AdvancedAnimator'      /tmp/boot.log   # 0
grep -ac 'FileNotFoundException' /tmp/boot.log   # 0
grep -ac 'NoSuchFileException'   /tmp/boot.log   # 0
```

Referencia de un arranque sano con los 56 mods:

| Contador | Valor |
|---|---|
| `SERVER STARTED` | 1 |
| `required mod not found` | 0 |
| `AdvancedAnimator` | 0 |
| `FileNotFoundException` / `NoSuchFileException` | 0 |
| mods cargados | 56 (conjunto identico al cliente) |
| `ERROR` | 56 (ruido conocido, ver abajo) |
| `WARN` | ~1610 (ruido conocido) |

### Ruido de log conocido (NO es un fallo)

- `AngelCodeFont failed to load page 0 texture`
- `XuiSkin Could not find icon`
- `IsoPropertyType.lookupOrDefaultStr`
- `InventoryItem.loadItem data length not matching`
- `Missing ThumpSound`
- `unknown option "-modfolders"`
- `"ADMINPASSWORD" variable is not set` (compose)

## Backups

En `/opt/backups/` del VPS:

| Artefacto | Que es |
|---|---|
| `vol-data-pre-recreacion-20261104-184840.tar.gz` | volume de datos (mundo + INI + DB) |
| `vol-workshop-pre-recreacion-20261104-211102.tar.gz` | volume del workshop (2,0 GB) |
| `backup-pre-recreacion-20261104-180721/` | DB (`ZomboidMGTSolutions.db`), INI, SandboxVars |
| `env-pre-recreate-<ts>` / `compose-pre-recreate-<ts>.yaml` | `.env` y compose previos |
| `zomboid-mods-watch.py.bak-<ts>` | watcher anterior |
| `zomboid_control.py` | libreria + CLI de control (RCON, docker, map toggles) |
| `bot-telegram.py` | bot v5 |
| `bot.env` | token del bot (secreto, no versionar) |

sha256 del snapshot del workshop:

```
2c987ea7c1fb3b706abeacd38424cd516ba724340b901da2878051e0e4cb32e1
```

## Comandos utiles en el VPS

```bash
# estado
sudo docker ps --filter "label=coolify.applicationId=8"

# logs
sudo docker logs --tail 100 <container>

# jugadores conectados (por RCON, loopback)
python3 /opt/backups/zomboid_control.py players

# toggles de mapa
python3 /opt/backups/zomboid_control.py map

# usuarios
python3 /opt/backups/zomboid_control.py users

# estado del watcher
sudo systemctl is-active zomboid-mods-watch.timer insforge-backup-bot.service
sudo journalctl -u zomboid-mods-watch.service -n 50 --no-pager
```

## Pendientes

- [ ] Regenerar el token de la API de Coolify (el boton Redeploy del bot
      devuelve `401`).
- [ ] Guardar la clave de la cuenta `admin` del server en un sitio seguro.
- [ ] Verificacion final con el cliente conectado: entrar al mundo nuevo y
      revisar el `console.txt` del cliente (`ERROR`, `AdvancedAnimator`,
      `WorldDictionary`).
