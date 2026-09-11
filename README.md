# DBPOSTGRES

> **Sin variables de entorno aprobadas por Kiro, este Power no funciona.** Kiro solo expande `${VARIABLE}` en el `mcp.json` cuando esa variable está explícitamente aprobada — si no, la conexión falla con un error de autenticación que no menciona nada sobre variables. Ver "Puesta en marcha" más abajo.

Un Kiro Power dedicado exclusivamente a **PostgreSQL**, usando el paquete MCP [`@microsoft/postgres-mcp`](https://www.npmjs.com/package/@microsoft/postgres-mcp), mantenido por Microsoft.

Separado a propósito de otros motores: un Power por servidor MCP, para que el panel de Kiro quede claro y cada uno se active de forma independiente. Si trabajás con varios motores, instalá también `dbmysql` y `dbmssql`, y activá el que necesites en cada momento — el que no está activo no consume contexto.

## Soporte y privacidad

**Soporte:** yoel.moreno.ym@gmail.com

**Política de privacidad:** [PRIVACY.md](./PRIVACY.md)

## ⚠️ Tres validaciones obligatorias antes de usarlo

Estas tres cosas tienen que estar en orden **antes de intentar cualquier consulta**. Son la causa de la gran mayoría de los fallos de conexión.

### 1. Node.js 22 o superior

Verificar en una terminal: `node -v` → debe mostrar `v22.x.x` o mayor. **Este paquete lo exige explícitamente** para correr vía `npx` — es el requisito más restrictivo de los tres Powers de base de datos. Si trabajás con varios proyectos que usan versiones distintas de Node, asegurate de que la correcta esté activa **en el momento en que arrancás Kiro** — el proceso fija la versión al arrancar, no la vuelve a leer después.

### 2. La variable de entorno creada

`PGPOWER_DSN` tiene que existir en el sistema y **en MAYÚSCULAS exactas**. Verificar **solo que exista, nunca su valor**, con `if defined PGPOWER_DSN (echo Definida) else (echo NO definida)` en `cmd.exe` — nunca con `echo %VAR%`. Esto importa especialmente acá: la variable es una cadena de conexión completa, así que imprimirla expone usuario, contraseña, host y base de una sola vez.

### 3. La variable aprobada en el IDE de Kiro

Confirmar en `Ctrl + ,` → **Mcp Approved Env Vars** que `PGPOWER_DSN` figure en la lista.

---

## Variable de entorno

A diferencia de los Powers de MySQL y SQL Server (que usan cinco variables separadas), este paquete toma **una sola cadena de conexión completa**:

| Variable del sistema | Variable interna del paquete |
|---|---|
| `PGPOWER_DSN` | `POSTGRES_MCP_CONNECTION_STRING` |

Formato (libpq URI): `postgresql://usuario:contraseña@host:puerto/base`

Si la contraseña tiene caracteres especiales (`@`, `:`, `/`, `#`, `?`), rompen el formato de URI y hay que codificarlos en porcentaje (`@` → `%40`, etc.).

## Sobre el mecanismo de este paquete

`@microsoft/postgres-mcp` normalmente guarda las conexiones en el **keyring del sistema operativo**, mediante su propia CLI, sin que la credencial pase por ningún archivo. Ese es su modo recomendado y el más seguro.

Este Power usa a propósito su **modo alternativo para CI/headless** (`POSTGRES_MCP_CONNECTION_STRING`), que crea un perfil de conexión implícito al arrancar. Se eligió así porque es lo que permite configurarlo con una variable de entorno, de forma consistente con los otros Powers de base de datos.

La contrapartida: en este modo la credencial vive en el entorno del proceso, no en el keyring. Si preferís la protección del keyring, podés usar la CLI del paquete directamente en vez de este Power.

## Solo-lectura: leé esto antes de apuntar a producción

**Este paquete no fuerza solo-lectura por defecto.** A diferencia del Power de MySQL (donde el servidor MCP bloquea escrituras por configuración), acá la protección depende de dos capas:

1. **Un rol de PostgreSQL con permisos `SELECT` únicamente.** Es la protección real y la recomendada — ninguna instrucción del agente puede sobrepasar un permiso del motor.
2. **Las reglas del skill `query-helper`:** confirmación explícita antes de cualquier escritura.

Si vas a apuntar a una base productiva, configurá el punto 1. La capa 2 sola no es una garantía.

## Qué NO hace (por decisión de diseño)

- **No lee ningún archivo que pueda contener credenciales.**
- **No lee ni genera `product.md`.**
- **No recomienda ni sugiere instalar otros Powers.**
- **No imprime nunca el valor de una variable de entorno**, ni siquiera para depurar.

## Instalación (a nivel de usuario, global para todos los proyectos)

Panel de **Powers** en Kiro → **Add Custom Power** → **Import power from a folder** o **Import power from GitHub**.

Los Powers se instalan a nivel de usuario por defecto (`C:\Users\<tu usuario>\.kiro\powers\installed\dbpostgres`, con el nombre en minúsculas porque el esquema de Agent Plugins solo admite minúsculas, dígitos, guiones y puntos en el campo `name`), y quedan disponibles en **todos los workspaces** sin reinstalar por repositorio.

## Puesta en marcha

1. **Definir la variable** en una terminal (`cmd.exe` en Windows):

   ```
   setx PGPOWER_DSN postgresql://tuusuarioaqui:tupasswordaqui@tuhostaqui:5432/tubasedatosaqui
   ```

   Verificar en una terminal **nueva** que exista, sin imprimir el valor:

   ```
   if defined PGPOWER_DSN (echo Definida) else (echo NO definida)
   ```

   **Importante:** usar `setx` sin la bandera `/M`, para que quede a nivel de **usuario** (`HKCU`), visible solo para tu perfil. Con `/M`, o definiéndola en "Variables del sistema" desde el Panel de Control, queda a nivel de **máquina** (`HKLM`) — visible para cualquier usuario de esa computadora. Como esta variable contiene la contraseña dentro de la cadena, nunca la guardes a nivel de máquina.

2. **Aprobarla en el IDE.** Al guardar el `mcp.json` (o al activar el Power por primera vez), debería aparecer una ventana emergente de aprobación. Si no aparece: `Ctrl + ,` → **Mcp Approved Env Vars** → agregar `PGPOWER_DSN`. Nota: `kiro-cli` tiene su propia lista aparte, en `~/.kiro/settings/cli.json`.

3. Si la variable se definió con Kiro ya abierto, **cerrar y reabrir el IDE por completo**.

4. **Activar el Power** — no hay ningún servidor que "habilitar" por separado: para un Agent Plugin, el servidor MCP se activa junto con el Power. Se activa solo cuando le pedís al agente algo de PostgreSQL, o manualmente desde **Powers** → `dbpostgres` → **Try power**.

El skill `setup` incluye el detalle completo y una tabla de diagnóstico.

## Protección de tablas grandes

Antes de consultar una tabla por primera vez, el Power estima su tamaño con `reltuples` de `pg_class` (nunca con un `COUNT(*)` sobre la tabla entera). Por debajo de cien mil filas consulta libremente. Entre cien mil y un millón exige al menos un filtro selectivo. Por encima de un millón la trata como **tabla protegida**: exige filtro selectivo, rango de fechas cerrado si aplica, índice verificado, `LIMIT` explícito, y prohíbe `SELECT *`, ordenar sin límite, uniones sin filtro y funciones sobre la columna filtrada.

Si hay duda sobre una consulta, revisa el plan con `EXPLAIN` — **nunca `EXPLAIN ANALYZE` sobre una tabla protegida**, porque en PostgreSQL ese sí ejecuta la consulta de verdad.

## Múltiples bases PostgreSQL

Este Power trae una sola conexión configurada. Si hace falta una segunda (por ejemplo, desarrollo y producción), se duplica el bloque `postgres` en el `mcp.json` del paquete con otro nombre de entrada, otra variable y otro `POSTGRES_MCP_PROFILE_NAME` — es un cambio del Power en sí, no algo que se alterne en tiempo de uso.

## Problemas conocidos

**Las variables `${VAR}` a veces no se expanden, incluso aprobadas y presentes en el entorno.** En un Power tipo Agent Plugin, las referencias `${VAR}` del bloque `env` pueden llegar literales al servidor MCP, aunque la variable esté aprobada y confirmada presente. Es un bug del runtime de Kiro, reportado y con reproducciones en varias superficies (CLI/Docker, stdio, Kiro Web, Agent Plugins). Si te pasa, probá en este orden:

1. Cerrar Kiro **por completo** (no solo recargar la ventana) y volver a abrirlo.
2. Confirmar que la variable esté a nivel de **usuario** (`HKCU`), no de máquina (`HKLM`).
3. Revisar si hay una actualización de Kiro pendiente.
4. Reinstalar el Power desde cero.

**Si después de todo eso sigue sin expandir, el último recurso es hardcodear el valor — pero solo en tu copia ya instalada, nunca en el paquete que publicás o compartís:**

- Editá el `mcp.json` de la carpeta **instalada** (`~/.kiro/powers/installed/dbpostgres/mcp.json` o el equivalente en Windows), reemplazando `"${PGPOWER_DSN}"` por la cadena real. El agente **nunca escribe el valor real por vos** — solo indica dónde y cómo hacerlo, con un placeholder de ejemplo:

  ```jsonc
  "env": {
    "POSTGRES_MCP_CONNECTION_STRING": "postgresql://tuusuarioaqui:tupasswordaqui@tuhostaqui:5432/tubasedatosaqui",
    "POSTGRES_MCP_PROFILE_NAME": "dbpostgres"
  }
  ```

- **Nunca** hagas ese cambio en la carpeta fuente que vas a publicar o subir a GitHub.
- Tratá esa copia con hardcode como un archivo de credenciales: no la compartas, no la subas a ningún repositorio, no la pegues en un chat.
- **Vas a tener que repetir el hardcode después de cada actualización del Power** (`Check for updates` sobrescribe el `mcp.json` instalado).
- Si llegás a este punto, conviene rotar esa credencial después, ya que estuvo en texto plano en un archivo en disco.

**El validador de esquemas de VS Code/Kiro puede bloquear `agent-plugins.org` por defecto**, con el error "Location is untrusted". Se resuelve agregando ese dominio a `json.schemaDownload.trustedDomains` en el `settings.json` de usuario (`Ctrl+Shift+P` → "Preferences: Open User Settings (JSON)").

## Actualizar el Power

Panel de Powers → `dbpostgres` → **Check for updates** → **Install updates**.

## Estructura

```
DBPOSTGRES/
├── plugin.json
├── mcp.json
├── README.md
├── PRIVACY.md
├── LICENSE
└── skills/
    ├── setup/
    │   └── SKILL.md
    └── query-helper/
        └── SKILL.md
```

## Servidor MCP usado

| Motor | Paquete |
|---|---|
| PostgreSQL | `@microsoft/postgres-mcp` |

Paquete de terceros con sus propios términos y licencia. Revisalo antes de usarlo contra bases productivas.

## Licencia

MIT
