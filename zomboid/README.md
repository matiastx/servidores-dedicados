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
>
> **2026-11-05 (tarde)**: **reglas internas de orden de carga de mods**
> (basadas en la guia de XGamingServer) integratedas en el boton
> **🎛️ MODS on/off**: el panel resume los problemas, avisa antes de activar
> un mod incompatible y ofrece **ARREGLAR** dejando una variante por grupo.
> Ver [Reglas internas de orden de carga de mods](#reglas-internas-de-orden-de-carga-de-mods).

## Resumen del estado actual

| Que | Valor | Donde vive |
|---|---|---|
| Mods cargados | **56** (los mismos que carga el cliente) | `Mods=` en el INI |
| Items de Workshop | **138** | `WorkshopItems=` en el INI |
| Mods disponibles (activables a mano) | **184** en el workshop del server | catalogo del bot |
| Overrides de mods (on/off manual) | ninguno por defecto | `/opt/backups/zomboid-mods-override.json` |
| Reglas internas de orden de carga | activas (resumen en el panel del bot) | `zomboid_control.py` |
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

### MODS on/off (botón del bot)

El botón **🎛️ MODS on/off** lista **los 184 mods que hay en el workshop del
server** con un check al lado, para activarlos o desactivarlos a gusto.

| Boton | Que hace |
|---|---|
| `pz_mods_panel` | abre el listado (paginado, 8 mods por pagina = 23 paginas) |
| `pz_mods_p:<n>` | cambia de pagina |
| `pz_tog:<hash>:<pag>` | activa/desactiva un mod (hash = sha1 del id, 10 hex) |
| `pz_mods_apply` | pide confirmacion para aplicar (expira a los 5 min) |
| `pz_ok_mods_apply` | aplica los cambios (stop -> INI -> start -> verificar) |
| `pz_mods_reset` | borra los overrides y vuelve a la base del cliente |

Callbacks de las reglas internas (ver
[Reglas internas de orden de carga de mods](#reglas-internas-de-orden-de-carga-de-mods)):

| Boton | Que hace |
|---|---|
| `pz_mods_val` | detalle de los problemas del orden de carga |
| `pz_mods_fix` | lista los grupos de mods que no pueden convivir |
| `pz_mods_fix_g:<clave>` | muestra las variantes de un grupo |
| `pz_mods_keep:<clave>:<hash>` | elige que variante conservar |
| `pz_ok_mods_fix:<clave>:<hash>` | confirma el arreglo (expira a los 5 min) |
| `pz_ok_mods_tog:<hash>:<pag>` | "⚠️ Activar igual" (2º paso, expira a los 5 min) |

La `clave` del grupo es `sha1(miembros ordenados)[:8]`: es **estable** entre
reinicios, asi que el mismo grupo siempre tiene el mismo boton.

El check muestra el estado **configurado**; un `🔄` marca los mods cuyo estado
difiere del que tiene el server ahora (cambio pendiente de aplicar).

**Como se guardan los cambios.** Cargar mods exige reiniciar el server, asi que
el bot **no** toca el INI al vuelo: guarda un override por mod en

```
/opt/backups/zomboid-mods-override.json     {"<mod_id>": "on" | "off"}
```

Relativo a la **base** (la referencia del cliente):

```
set efectivo = (base UNION "on") - "off"
```

- El override se **borra** solo cuando el estado pedido coincide con el de la
  base (asi el JSON queda lo mas chico posible).
- El boton **Aplicar** reescribe `Mods=` / `WorkshopItems=` y reinicia.
  `WorkshopItems=` conserva **todos** los items: desactivar un mod no
  desinstala su contenido, asi que volverlo a activar es instantaneo.
- El boton **Volver a la base** borra los overrides (no reinicia nada).
- El sync **🧩 MODS = Cliente** respeta los overrides: el target es el set
  efectivo, no la base pelada.

**Orden de carga.** Se conserva el orden actual del server; los mods apagados
desaparecen y los activados a mano se insertan con la posicion que les da la
regla: **despues de su requisito activo**, si no tienen ninguno **al final**,
y si eso deja varias posiciones validas se desempata por **categoria**
(framework -> mapa -> vehiculo -> arma/item -> building/crafting -> UI ->
overhaul, el orden de la guia de XGamingServer). Si el cliente impone otro
orden, se adopta el de la **capa 2** (`f:0 st:0`) — la *union* arranca por la
capa local, asi que usar sus posiciones pondria los mods en otro lugar.

> No se reordena lo que ya esta cargado: el `Mods=` del server tiene que
> seguir siendo el de la **capa 2** del cliente o los jugadores ven distinto
> contenido. La guia se usa solo como criterio de desempate al insertar algo
> nuevo.

**Dependencias.** El bot escanea el `mod.info` de cada mod del workshop (el de
la subcarpeta de version, que es el que carga B42) y avisa:

- al **activar** un mod, que requisitos suyos no van a estar activos;
- al **desactivar** un mod, cuantos mods activos dependen de el.

> `damnlib` es requisito de **93** mods de vehiculos y **no esta en el
> workshop**: activarlos sin el no va a funcionar. El bot lo avisa.

**Catalogo.** Se arma escaneando
`<volumen workshop>/content/108600/<item>/mods/<mod>/**/mod.info`. En el server
hay **138 items** y **184 mods unicos** (todos los 56 del `Mods=` estan
incluidos). El escaneo tarda ~0.1 s y se cachea 5 min.

Verificado el 2026-11-05 end-to-end en el VPS: se desactivo `87cruiser`, se
aplico (el server reinicio y cargo **55/55** mods), se reseteo y se volvio a
aplicar (**56/56**). El orden del `Mods=` quedo identico a la capa 2.

### Reglas internas de orden de carga de mods

Reglas propias del server, derivadas a fondo de la guia de
[XGamingServer](https://xgamingserver.com/docs/project-zomboid/mod-load-order),
que se respetan al **agregar, quitar, activar o desactivar** mods. Viven en
`zomboid_control.py` y estan integratedas en el boton **🎛️ MODS on/off**.

**De donde salen las reglas, por prioridad:**

1. **Campos nativos del juego** (leidos del `mod.info`): `require=` (el
   requisito carga antes), `loadmodafter=` / `loadafter=`, `incompatible=` (no
   pueden convivir), `pack=` (grupo excluyente: maximo uno activo) y
   `versionmin=` (version minima del juego que soporta el mod).
2. **Grafo de archivos**: que mods pisan los mismos archivos. Se calcula con la
   misma resolucion de carpetas que usa el juego. El juego carga en orden y
   **gana el ultimo** del `Mods=`.
3. **Guia de XGamingServer** (framework -> mapa -> vehiculo -> arma/item ->
   building/crafting -> UI -> overhaul): se usa **solo como desempate** al
   insertar un mod nuevo. **No** reordena lo que ya esta cargado.

> **Desviacion documentada de la guia.** La guia pide mantener `Mods` y
> `WorkshopItems` en el mismo orden. En B42 **no aplica**: son namespaces
> distintos (`Mods` = ids de mod en orden de carga; `WorkshopItems` = ids del
> Workshop a descargar). Hoy hay **56 mods y 138 items**: la correspondencia
> 1:1 es imposible. El juego usa `Mods` para el orden y `WorkshopItems` solo
> para que se descargue el contenido.

**Modelo de resolucion de archivos** (verificado contra el log real del server:
*winners 354/354 correctos, 0 proveedores perdidos*):

```
contenido(mod) = <mod>/common/**
              + <mod>/<version>/common/**
              + <mod>/<version>/**   (excluyendo el subdirectorio common/)
<version> = carpeta numerica mas alta <= version del juego (42.21)
```

> **Bug encontrado y corregido el 2026-11-05**: el metadata de los mods unia
> el `require`/`incompatible` de **todas** las carpetas de version. Eso metia
> dependencias falsas del `mod.info` raiz de B41 (`TombBodyCustom` pedia
> `SpnCharCustom`, que no existe; la 42.13 que carga el juego pide `SPNCC`) y
> subia el conteo de incompatibles. Ahora la fuente primaria es **la carpeta
> de version que carga el juego**; el `versionmin` si se toma como el maximo
> de todas (por eso `CleanUI` avisa que pide 42.22).

**Estado real del server** (medido el 2026-11-05, 56 mods, juego 42.21.0):

| Metrica | Valor |
|---|---|
| Incompatibles activos (pares unicos) | **41** (56 pares dirigidos) |
| Pack `Seasons` con 5 variants activos | **1** (deberia haber 1) |
| Violaciones de `require` / `loadmodafter` | **0** |
| Pares que se pisan archivos | **103** (ganador vs perdedor) · **240** (todos vs todos) |
| Paths en choque | **355** |
| `versionmin` > version del juego | **CleanUI** (pide 42.22, corre 42.21) |
| `require` inexistentes | `damnlib` (93 mods de vehiculos), `SM4BootsExpandedB42`, `SM4BootsExpandedB42VanillaONLY` |

> El juego **no** aplica `incompatible=`: los mods incompatibles se cargan
> igual y solo se pisan archivos (gana el ultimo del `Mods=`). Por eso la regla
> es **avisar**, no bloquear en runtime.
>
> `CleanUI` (pide juego 42.22, corre 42.21) se **marca y avisa**: no se
> desactiva solo, pero **bloquea su activacion** mientras el server no se
> actualice.

**Que hace el bot:**

- El panel **🎛️ MODS** muestra el **resumen** (criticos / altos / medios) y
  agrega los botones **🔎 Problemas** y **🛠 ARREGLAR**.
- **🔎 Problemas** (`pz_mods_val`) lista los hallazgos por severidad, con los
  problemas estructurales primero (packs, incompatibilidades, dependencias).
- **Activar** un mod con problema **critico** (dependencia faltante,
  incompatible activo, pack excluyente) **no lo activa**: muestra los motivos
  y pide un **2º paso explicito** ("⚠️ Activar igual", expira a los 5 min).
- **Desactivar** avisa cuantos mods activos dependen de el, pero no bloquea.
- **🛠 ARREGLAR** (`pz_mods_fix`) lista los grupos de conflicto; al elegir uno
  muestra sus variantes y **exactamente que cambia** antes de aplicar
  (incluye el aviso de jugadores conectados). Deja **una variante por grupo**.
- La validacion se cachea 120 s en el bot; en el server validar tarda
  **0.04 s**.

En el server hay **2 grupos de conflicto** hoy: el pack `Project Seasons` (5
variantes activas) y el grupo de texturas del cuerpo (`SPNRetexture*` /
`TombBody*` / `SPNCC*`). La propuesta por defecto para Seasons es
`Project_Seasons_B42`.

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
| **🎛️ MODS on/off** | listado de los 184 mods del server con check para activar/desactivar + reglas de carga (ver [MODS on/off](#mods-onoff-boton-del-bot)) |
| **👥 Usuarios** | submenu con botones: listado -> usuario -> eliminar / cambiar rol |
| **🗺️ Mapa** | los tres toggles de la tabla de arriba |
| **Aplicar todas** | reinicia aplicando las updates de Steam |
| **Reiniciar** / **Apagar** / **Encender** | control del contenedor |
| **Redeploy** | redeploy via la API de Coolify |
| **Log** | ultimas lineas del server |

El submenu de **Usuarios** es a botones: se muestra el listado (un boton por
usuario), al tocar uno aparecen **🔑 Cambiar rol** (un boton por rol) y
**🗑️ Eliminar usuario**, y al confirmar se ejecuta por RCON. Los comandos
`/pz alta`, `/pz borrar` y `/pz rol` siguen funcionando.

El submenu de **MODS on/off** es a botones tambien: pagina de 8 mods con
`✅`/`❌`, tocá uno para alternarlo, y despues **✅ Aplicar cambios** (reinicia)
o **↩️ Volver a la base** (descarta los overrides). El comando `/pz mods on/off`
abre el mismo listado.

El mismo panel tiene el resumen de las **reglas de carga** y los botones
**🔎 Problemas** y **🛠 ARREGLAR**: activar un mod con conflicto critico pide
un 2º paso ("⚠️ Activar igual") y **ARREGLAR** deja una variante por grupo,
mostrando que cambia antes de aplicar. Ver
[Reglas internas de orden de carga de mods](#reglas-internas-de-orden-de-carga-de-mods).

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
(`ops-deploy-recreate.sh`, `ops-restore-whitelist.sh`, `ops-reapply-ini.sh`,
`ops-final-check.sh`, `ops-update-coolify-db.sh`). No se versionan: quedan
solo en la PC.

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
