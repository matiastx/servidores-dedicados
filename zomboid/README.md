# Server de Project Zomboid

Runbook operativo del server dedicado. Imagen:
[`danixu86/project-zomboid-dedicated-server`](https://github.com/Danixu/project-zomboid-server-docker).

## Arranque

El primer arranque crea el mundo y **exige** `ADMINPASSWORD` en el panel: si falta,
el entrypoint falla porque el server pide la clave por consola.

```bash
# en el VPS
sudo docker logs -f zomboid
```

Señales de arranque sano:

```
*** INFO: Server is listening on port 16261
```

El mundo se crea en `/home/steam/Zomboid/Server/<SERVERNAME>/` (storage de Coolify).

**Apenas el mundo esté creado, borrar `ADMINPASSWORD` del panel y redeployar**: el
entrypoint lo pasa como argumento del server y queda en claro en los logs.

## Puertos

| Puerto | Protocolo | Para qué |
|---|---|---|
| 16261 | UDP | puerto principal del juego (`PORT`) |
| 16262 | UDP | `UDPPORT`, conexión directa (`UDPPORT`) |
| 8766 / 8767 | UDP | cliente de Steam |
| 16262-16272 | TCP | clientes sin Steam (1 por slot + 1) |

Con 8 slots el rango TCP es `16262-16272`. Si se sube la cantidad de jugadores,
ampliar el rango (y su port forwarding) al mismo tiempo.

## Jugadores

- Entrar por **IP:puerto** (`181.164.89.26:16261` desde internet, `100.75.207.93:16261`
  desde Tailscale). Project Zomboid no usa dominios para conectarse.
- Con `PUBLIC=true` el server aparece en el navegador del juego.
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

El storage `/home/steam/Zomboid` es lo único que hay que respaldar (mundo + INIs).
Copiar en caliente puede dar un mundo inconsistente: detener el container, copiar,
levantar.

```bash
# en el VPS
sudo docker stop zomboid
sudo tar czf /tmp/zomboid-$(date +%F).tar.gz -C /data/coolify/applications/<uuid>/<storage> .
sudo docker start zomboid
```

En Coolify no hay S3 configurado, así que el destino del backup hay que elegirlo
(rsync a otra maquina, S3 externo, etc.).

## Diagnóstico

```bash
sudo docker ps --filter name=zomboid          # estado
sudo docker logs --tail 100 zomboid           # logs
sudo ss -ulpn | grep 16261                    # puerto escuchando
sudo docker port zomboid                      # puertos publicados
```

Si el server no aparece en el navegador del juego: `PUBLIC=true`, puerto 16261/UDP
abierto en el router, y `STEAMVAC=True` para servidores de Steam.

## Memoria y CPU

El VPS tiene 4 vCPU y 8 GB de RAM (compartidos con Coolify e InsForge).
`MEMORY=4096m` + límite de Coolify `6g` deja headroom. Si aparecen OOM kills en el
host, bajar `MEMORY` a `3072m`.
