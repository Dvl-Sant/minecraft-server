# Minecraft Server (Paper) — Infraestructura como Código

Despliegue de un servidor de **Minecraft Paper** usando **Docker** (`itzg/minecraft-server`) y **Dokploy**, gestionado como Infraestructura como Código (IaC): cada `git push` a `main` despliega/actualiza el servidor automáticamente.

---

## 📂 Estructura del repositorio

```
.
├── docker-compose.yml   # Definición del servicio, volúmenes, puertos y variables
└── .gitignore           # Excluye datos del mundo, *.jar, secretos y logs
```

> Los datos del mundo (`/data`) viven en un **volumen nombrado** de Docker gestionado por Dokploy, **fuera** del repositorio. Esto garantiza persistencia estricta entre despliegues y habilita backups automáticos.

---

## ⚙️ Configuración principal

Toda la configuración no-secreta se declara como `environment` en `docker-compose.yml`:

| Variable | Valor por defecto | Descripción |
|---|---|---|
| `EULA` | `TRUE` | **Obligatorio.** Acepta el EULA de Mojang. |
| `TYPE` | `PAPER` | Tipo de servidor (`VANILLA`, `PAPER`, `FABRIC`, `FORGE`). |
| `VERSION` | `LATEST` | Versión de MC (`1.21.1`, `1.20.4`, ...). |
| `MEMORY` | `3G` | Heap JVM (`-Xms`/`-Xmx`). |
| `mem_limit` | `4g` | Tope duro de RAM del contenedor (≈4G totales). |
| `TZ` | `America/Guayaquil` | Zona horaria (logs/backups). |
| `ONLINE_MODE` | `TRUE` | `FALSE` para cuentas no-premium (offline). |
| `MAX_PLAYERS` | `20` | Máximo de jugadores. |

Edita estos valores, haz commit y push: Dokploy aplicará los cambios solo.

### Plugins (estrategia IaC, sin binarios en git)

Los plugins se declaran como variables de entorno y se **descargan automáticamente** en cada arranque:

```yaml
environment:
  MODRINTH_PROJECTS: "luckperms,essentialsx"   # slugs de Modrinth
  SPIGET_RESOURCES: "28140"                    # IDs de recursos de SpigotMC
  PLUGINS: "https://ejemplo.com/plugin.jar"    # URL directa a un .jar
```

> Deja estas líneas comentadas si aún no usas plugins.

---

## 🚀 Despliegue en Dokploy

1. Sube este repositorio a GitHub (rama `main`).
2. En Dokploy: **Proyecto → Nuevo servicio → Compose → tipo "Docker Compose"**.
3. **Provider:** GitHub → selecciona el repositorio y la rama `main`.
4. **Compose Path:** `./docker-compose.yml`.
5. Pestaña **General** → activa **Auto Deploy** (cada `git push` redeployará).
6. *(Opcional)* Pestaña **Environment** → variables sensibles (ej. `RCON_PASSWORD`).
7. Pulsa **Deploy** y espera al primer arranque (genera el mundo: ~3 min).

### Acceso de jugadores

- Conexión: `IP_DE_TU_SERVIDOR:25565`
- Abre el puerto **25565/TCP** en el firewall de tu proveedor cloud.
- La consola OP está disponible vía `docker attach <contenedor>` (gracias a `tty: true`).

---

## 💾 Backups

El volumen nombrado `minecraft-data` es compatible con **Volume Backups** de Dokploy (a S3).
Configúralo en Dokploy: pestaña **Backups / Volume Backups** del servicio.

---

## 🔄 Operaciones comunes

| Acción | Cómo |
|---|---|
| Cambiar versión de MC | Edita `VERSION`, commit, push. |
| Añadir plugins | Descomenta `MODRINTH_PROJECTS`/`SPIGET_RESOURCES`, commit, push. |
| Dar OP a alguien | Edita `OPS` con sus UUIDs, o usa la consola: `op <jugador>`. |
| Subir RAM | Aumenta `MEMORY` y `mem_limit` manteniendo ~1G de margen para la JVM. |
| Forzar redeploy | Cualquier `git push` a `main`, o botón Deploy en Dokploy. |

---

## ⚠️ Notas importantes

- **No uses rutas absolutas ni relativas al repo** para montar datos: Dokploy borra el directorio del repo en cada Auto Deploy (hace `git clone` limpio). Por eso se usa un volumen nombrado.
- **No se fijó `container_name`**: es requisito de Dokploy para que logs y métricas funcionen.
- **El tráfico de Minecraft no es HTTP**: el puerto `25565` se publica directo, sin pasar por Traefik/Dominios de Dokploy.
