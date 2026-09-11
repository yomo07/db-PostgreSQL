---
name: setup
description: Configurar dbpostgres para conectarse a PostgreSQL, definiendo y aprobando la variable de entorno correcta.
---

# Configurar dbpostgres

Este Power maneja **un solo motor** (PostgreSQL) con **un solo paquete MCP** (`@microsoft/postgres-mcp`, mantenido por Microsoft).

## Paso 0 (obligatorio, antes de cualquier consulta): las tres validaciones

Antes de ejecutar cualquier consulta o explorar el esquema, el agente debe confirmar estas tres cosas. No asumir que ya están listas solo porque el Power está instalado.

1. **Node.js 22 o superior.** Correr `node -v` y confirmar que devuelve `v22.x.x` o mayor. Este paquete lo exige explícitamente para correr vía `npx`. Si trabajás con otros proyectos que usan una versión distinta de Node, asegurate de que la correcta esté activa **en el momento en que arrancás Kiro** — el proceso fija la versión al iniciar, no la relee después. Este es el requisito más restrictivo de los tres Powers de base de datos.

2. **La variable de entorno creada.** Verificar únicamente su **presencia**, nunca su valor. El agente no debe correr ni sugerir ningún comando que imprima el contenido de una variable — ni `echo %VAR%`, ni `$env:VAR` a secas, ni nada equivalente. En Windows:

```
if defined PGPOWER_DSN (echo Definida) else (echo NO definida)
```

En PowerShell: `if ($env:PGPOWER_DSN) { "Definida" } else { "NO definida" }`. Ninguno revela el valor.

3. **Aprobada en Kiro.** Preguntar al usuario si ya aprobó `PGPOWER_DSN` en `Ctrl+,` → **Mcp Approved Env Vars**, o si vio la ventana emergente de aprobación. Una variable sin aprobar produce un error de autenticación que no menciona nada sobre esto.

**Cuándo repetir esta verificación:** al menos una vez al empezar una sesión de trabajo. Si una consulta falla con error de conexión o autenticación, volver a este Paso 0 antes de reintentar, en vez de repetir la misma consulta a ciegas.

## Regla absoluta sobre credenciales

**El agente NUNCA abre el contenido de un archivo que pueda contener credenciales.** Eso incluye `.env` y variantes, `application.properties`, `application.yml`, `dev.properties`, `local.properties`, `settings.py`, `appsettings.json`, `database.yml`, `pgpass`, `.pgpass`, y cualquier archivo con `secret`, `credential`, `password` o `key` en el nombre. Puede listar sus nombres; no los abre ni para diagnosticar un fallo de conexión.

Las credenciales viven únicamente como **variable de entorno del sistema**. Cuando el usuario las tenga dentro de un archivo del proyecto, el agente **no las lee ni las migra**; le explica el mecanismo y lo deja actuar.

**El agente nunca sugiere ni ejecuta un comando que imprima el valor de una variable de entorno.** Toda verificación se hace con comandos que confirman presencia sin revelar contenido. Aplica incluso si el usuario pide explícitamente ver el valor para depurar. Esto es especialmente importante acá: la variable es una cadena de conexión completa, así que imprimirla expone usuario, contraseña, host y base de una sola vez.

**Cualquier comando de ejemplo usa placeholders inequívocamente falsos.** El agente jamás inventa ni sugiere un valor concreto en nombre del usuario — solo valida que la variable exista, nunca manipula ni propone su contenido.

## Paso 1: Definir la variable de entorno

A diferencia de los Powers de MySQL y SQL Server (que usan cinco variables separadas), este paquete toma **una sola cadena de conexión completa**:

| Variable del sistema | Variable interna del paquete |
|---|---|
| `PGPOWER_DSN` | `POSTGRES_MCP_CONNECTION_STRING` |

Formato (libpq URI): `postgresql://usuario:contraseña@host:puerto/base`

En Windows:

```
setx PGPOWER_DSN postgresql://tuusuarioaqui:tupasswordaqui@tuhostaqui:5432/tubasedatosaqui
```

`setx` la deja permanente; `set` solo dura la terminal actual. Verificar en una terminal **nueva** que exista, sin imprimir el valor:

```
if defined PGPOWER_DSN (echo Definida) else (echo NO definida)
```

**Usar el nombre en MAYÚSCULAS exactas.** Windows no distingue mayúsculas para usar la variable, pero la lista de aprobación de Kiro compara el nombre como texto exacto.

**Importante:** usar `setx` sin la bandera `/M`, para que la variable quede a nivel de **usuario** (`HKCU`), visible solo para tu perfil. Con `/M`, o definiéndola en "Variables del sistema" desde el Panel de Control, queda a nivel de **máquina** (`HKLM`) — visible para cualquier usuario de esa computadora. Como esta variable contiene la contraseña dentro de la cadena, nunca la guardes a nivel de máquina.

**Si la contraseña tiene caracteres especiales** (`@`, `:`, `/`, `#`, `?`), rompen el formato de URI. Hay que codificarlos en porcentaje (`@` → `%40`, `:` → `%3A`, `/` → `%2F`, `#` → `%23`). El agente puede explicar esta regla, pero **no debe pedir ni manipular la contraseña real** para hacer la conversión — es el usuario quien la codifica en su editor.

## Nota sobre el mecanismo de este paquete

`@microsoft/postgres-mcp` normalmente guarda las conexiones en el **keyring del sistema operativo**, mediante su propia CLI (`connection add` + `connection set-password`), sin que la credencial pase por ningún archivo de configuración. Ese es su modo recomendado y el más seguro.

Este Power usa a propósito su **modo alternativo pensado para CI/headless**: la variable `POSTGRES_MCP_CONNECTION_STRING` crea un perfil de conexión implícito al arrancar, junto con `POSTGRES_MCP_PROFILE_NAME` para nombrarlo (ya viene fijo en el `mcp.json`, no hace falta definirlo). Se eligió así porque es lo que permite configurar el Power con una variable de entorno, de forma consistente con los otros Powers de base de datos.

La contrapartida a tener presente: en este modo la credencial vive en el entorno del proceso, no en el keyring. Si preferís la protección del keyring, podés usar la CLI del paquete directamente en vez de este Power.

## Solo-lectura

Este paquete **no fuerza solo-lectura por defecto** — los perfiles creados permiten escrituras salvo que se especifique lo contrario. A diferencia del Power de MySQL (donde el servidor bloquea escrituras por configuración), acá la protección depende de dos capas que el usuario debe establecer:

1. **Un rol de base de datos solo-lectura.** Es la protección real y la recomendada: conectarse con un usuario de PostgreSQL que solo tenga permisos `SELECT`. Ninguna instrucción del agente puede sobrepasar un permiso del motor.
2. **Las reglas del agente** (skill `query-helper`): confirmar explícitamente antes de cualquier escritura.

El agente debe **recomendar activamente el punto 1** cuando el usuario configure este Power por primera vez, sobre todo si la base es productiva.

## Paso 2 (obligatorio en el IDE): Aprobar la variable

Por seguridad, el IDE de Kiro solo expande variables de entorno explícitamente aprobadas. Al guardar el `mcp.json`, debería aparecer una ventana emergente de aprobación; si no aparece, `Ctrl + ,` → **Mcp Approved Env Vars** → agregar `PGPOWER_DSN`.

`kiro-cli` tiene su propia lista aparte, en `~/.kiro/settings/cli.json`; aprobar en un lado no aprueba en el otro.

## Paso 3: Reiniciar Kiro si la variable se definió con el IDE ya abierto

El proceso de Kiro fija su entorno al arrancar. Si corriste `setx` con Kiro ya abierto, no va a ver la variable nueva hasta cerrarlo y volverlo a abrir por completo.

## Paso 4: Activar el Power

Para un Power tipo Agent Plugin, el servidor MCP **no tiene un interruptor propio** de encendido/apagado — se activa y desactiva junto con el Power.

Dos formas: **automática**, cuando le pedís al agente algo relacionado con PostgreSQL en el chat; o **manual**, desde el panel de **Powers** → `dbpostgres` → **Try power**.

Si trabajás con varios motores, tené instalados los Powers correspondientes (`dbmysql`, `dbmssql`) y activá el que necesites en cada momento. Cada uno es independiente: el que no está activo no consume contexto.

## Diagnóstico

| Síntoma | Causa probable |
|---|---|
| Error de autenticación con credenciales correctas | Variable no aprobada en el IDE (Paso 2), o definida con Kiro ya abierto (Paso 3) |
| Aparece `${PGPOWER_DSN}` literal en un error, y ya verificaste nombre, aprobación y reinicio completo | Bug conocido del runtime de Kiro (ver README, "Problemas conocidos") |
| El servidor no arranca | Node.js activo al arrancar Kiro es menor a 22 — este paquete lo exige |
| Cero herramientas aunque el Power está instalado | Probar **Try power** desde el panel de Powers |
| La conexión falla y la contraseña tiene `@`, `:`, `/` o `#` | Esos caracteres rompen el formato URI; hay que codificarlos en porcentaje (Paso 1) |
| Error de permisos en una tabla | Permisos del rol de PostgreSQL, no del Power; revisar los `GRANT` de ese usuario |
| El agente propone abrir `.env` o `.pgpass` | Comportamiento prohibido; las credenciales van solo en la variable de entorno |
