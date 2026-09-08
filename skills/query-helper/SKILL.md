---
name: query-helper
description: Consultar PostgreSQL de forma segura, trayendo al contexto solo los datos necesarios.
---

# Consultar PostgreSQL con dbpostgres

Aplica siempre que se vaya a ejecutar una consulta.

## Antes de la primera consulta de la sesión: las tres validaciones obligatorias

No ejecutar ninguna consulta sin haber pasado por esto al menos una vez en la conversación (detalle completo en el skill `setup`, Paso 0):

1. Node.js 22 o superior (`node -v`) — requisito estricto de este paquete.
2. La variable `PGPOWER_DSN` creada en el sistema.
3. Aprobada en Kiro (`Ctrl+,` → Mcp Approved Env Vars).

Si una consulta falla con error de conexión o autenticación, volver a esta verificación antes de reintentar — no repetir la misma consulta a ciegas esperando que funcione distinto.

## Reglas de seguridad (no negociables)

1. **Solo-lectura por defecto.** Antes de cualquier `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `DROP`, `ALTER` o DDL, confirmar explícitamente con el usuario en el chat, incluso si ya autorizó una escritura antes en la misma conversación. **A diferencia del Power de MySQL, acá el servidor MCP no bloquea escrituras por configuración** — la protección real es un rol de PostgreSQL con permisos `SELECT` únicamente. Si el usuario no lo tiene configurado y la base es productiva, recomendárselo.
2. **Nunca leer archivos de credenciales**, ni siquiera para diagnosticar fallos de conexión. Incluye `.env` y variantes, `application.properties`, `application.yml`, `appsettings.json`, `database.yml`, `pgpass`, `.pgpass`, y cualquier archivo con `secret`, `credential`, `password` o `key` en el nombre. Se pueden listar sus nombres, nunca abrir su contenido.
3. **Nunca leer ni generar `product.md`.**
4. **Nunca recomendar ni sugerir instalar otros Powers.**
5. **Nunca sugerir ni ejecutar comandos que impriman el valor de una variable de entorno** (`echo %VAR%`, `echo $VAR`, etc.). Verificar solo presencia (`if defined VAR`), nunca contenido. Esto importa especialmente acá: `PGPOWER_DSN` es una cadena de conexión completa, así que imprimirla expone usuario, contraseña, host y base de una sola vez.

## Traer al contexto solo lo necesario

- **Esquema antes que datos.** Consultar `information_schema.columns` o los catálogos `pg_catalog` antes de traer filas de muestra.
- **Agregados antes que filas crudas.** Si la pregunta es "cuántos", "cuál es el máximo", "qué valores distintos existen", usar `COUNT`, `MAX`, `DISTINCT` — no traer el conjunto completo y contarlo después.
- **Columnas explícitas, nunca `SELECT *`.**
- **`LIMIT` siempre presente** al inspeccionar datos de ejemplo (5 a 10 filas bastan).
- No repetir consultas ya respondidas en la misma conversación.
- No consultar tabla por tabla si una consulta al catálogo responde todo de una vez.
- No volcar resultados grandes al chat; resumir.

## Tablas grandes: medir antes de consultar (regla dura)

Una consulta mal acotada contra una tabla de muchos millones de filas no es solo lenta: puede degradar la base para todos los demás, mantener locks abiertos y llegar a dejarla pegada. Esta regla protege la base, no el contexto, y por eso está por encima de cualquier conveniencia.

### Paso 1: clasificar la tabla antes de la primera consulta

Antes de consultar una tabla por primera vez, estimar su tamaño con `reltuples` de `pg_class` (junto con `pg_total_relation_size` si interesa el peso en disco). Nunca con un `COUNT(*)` sobre la tabla entera — esa es justamente la consulta que se quiere evitar.

`reltuples` es una **estimación** que mantiene el planificador, no un conteo exacto: puede estar desactualizada si la tabla no pasó por `ANALYZE` recientemente, y vale `-1` en tablas nunca analizadas. Aproximado alcanza para clasificar; no reportarlo como cifra exacta.

### Paso 2: aplicar el umbral

| Tamaño estimado | Comportamiento |
|---|---|
| Menos de cien mil filas | Consultar libremente, con columnas explícitas y `LIMIT` |
| Cien mil a un millón | Exigir al menos un filtro selectivo y `LIMIT` obligatorio |
| Más de un millón | **Tabla protegida.** No consultar sin filtro |

Guardar la clasificación en la conversación y no volver a medir la misma tabla dos veces.

### Paso 3: reglas para tablas protegidas

1. **Exigir un filtro selectivo:** clave primaria, número de cuenta, identificador de usuario o código de tipo.
2. **Exigir un rango de fechas cerrado** si la tabla tiene columna temporal (con inicio y fin, nunca abierto hacia atrás) — es el filtro más importante en tablas de movimientos, transacciones o auditoría.
3. **Verificar que el filtro use un índice.** Consultar `pg_indexes` o `pg_stat_user_indexes` antes de asumir que un filtro es eficiente.
4. **`LIMIT` obligatorio y explícito.**
5. **Nunca `SELECT *`.**
6. **Nunca `ORDER BY` sin `LIMIT`.**
7. **Nunca `JOIN` entre dos tablas protegidas** sin filtro selectivo en ambas.
8. **Nunca funciones sobre la columna filtrada** (por ejemplo `EXTRACT(YEAR FROM fecha) = 2025`): anula el índice salvo que exista un índice funcional específico. Comparar la columna cruda contra un rango.
9. **Nunca `SELECT ... FOR UPDATE`** ni nada que tome locks de escritura.

### Cómo pedir los filtros

Al negarse, ser concreto y ofrecer salida: decir el tamaño aproximado, pedir lo que falta, y proponer alternativas (agregados por período, una tabla de resumen si existe, una muestra acotada a un solo día).

### Verificar el plan cuando haya duda

Si el usuario insiste en una consulta que parece riesgosa, correr `EXPLAIN` antes de ejecutarla. Si el plan muestra un `Seq Scan` sobre una tabla protegida o un costo desproporcionado, informarlo y proponer el ajuste antes de ejecutar.

**Nunca usar `EXPLAIN ANALYZE` sobre una tabla protegida:** a diferencia de `EXPLAIN` a secas, `ANALYZE` **ejecuta la consulta de verdad**. Esta distinción es crítica en PostgreSQL.

## Particularidades de PostgreSQL a tener presentes

- **Esquemas.** Una base puede tener varios esquemas además de `public`. Si una tabla no aparece, puede estar en otro esquema — revisar `information_schema.tables` sin filtrar por `table_schema = 'public'` antes de concluir que no existe.
- **Identificadores en minúsculas.** PostgreSQL pasa los identificadores sin comillas a minúsculas. Una tabla creada como `"MiTabla"` (con comillas) solo se consulta con comillas dobles; sin ellas, busca `mitabla`.
- **Vistas materializadas.** `pg_matviews` las lista aparte de las tablas normales; pueden ser una alternativa liviana a consultar una tabla protegida.

## Verificar antes de asumir

Nunca inventar nombres de tablas, esquemas o columnas; confirmarlos contra `information_schema` o los catálogos de `pg_catalog` primero.
