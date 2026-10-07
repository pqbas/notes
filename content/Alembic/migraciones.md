---
title: Aplicar migraciones de base de datos con Alembic
tags:
  - alembic
  - database
  - python
---

Alembic versiona el esquema como una cadena de revisiones. Cada una sabe subir
y bajar, y la tabla `alembic_version` guarda en cuál está parada la base.

## 1. Respaldo

Siempre antes de cualquier comando de escritura. Un `upgrade` que transforma
columnas no siempre se revierte limpio.

```bash
cp data/robot/robot.db "data/robot/robot.db.bak-pre023-$(date +%Y%m%d-%H%M%S)"
```

En PostgreSQL:

```bash
pg_dump -Fc -f backup-pre023.dump "$DATABASE_URL"
```

## 2. Revisión actual

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini current
```

Devuelve el identificador, por ejemplo `022`. Si sale vacío, la base nunca fue
migrada.

| Comando                  | Qué muestra                                     |
| ------------------------ | ----------------------------------------------- |
| `alembic current`        | la revisión de la base ahora mismo              |
| `alembic heads`          | la última revisión disponible en el código      |
| `alembic history`        | la cadena completa, de la más nueva             |
| `alembic history -r022:` | solo lo pendiente desde `022`                   |

Si `current` y `heads` coinciden no hay nada pendiente. Si `heads` devuelve más
de una línea hay ramas sin fusionar: `alembic merge` antes de migrar.

## 3. Aplicar

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini upgrade head
```

La salida nombra cada salto: `Running upgrade 022 -> 023`.

| Elemento                  | Para qué                                          |
| ------------------------- | ------------------------------------------------- |
| `ENV_FILE=.env.robot`     | elige a qué base apunta                           |
| `-c src/back/alembic.ini` | ubica el ini cuando no está en la raíz            |
| `uv run`                  | ejecuta dentro del entorno del proyecto           |

El `ENV_FILE` explícito hace falta porque `make db-migrate` tiene `.env.server`
fijo en el target. En el robot hay que invocar Alembic directo, o la migración
cae sobre la base equivocada.

Para avanzar de a un paso cuando hay varias pendientes:

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini upgrade +1
```

## 4. Verificar

```bash
# quedó en la revisión esperada
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini current

# el esquema cambió
sqlite3 data/robot/robot.db ".schema sessions"

# los datos siguen ahí
sqlite3 data/robot/robot.db "SELECT COUNT(*) FROM sessions;"
```

Comparar el conteo antes y después distingue una migración que solo alteró el
esquema de una que tocó datos sin querer.

## 5. Volver atrás

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini downgrade -1
```

Solo funciona si la revisión implementó su `downgrade`. Muchas lo dejan en
`pass`: el comando no falla y tampoco deshace nada. La garantía real es el
respaldo del paso 1.

```bash
cp data/robot/robot.db.bak-pre023-20260910-025000 data/robot/robot.db
```

## 6. Crear una revisión

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini revision \
  --autogenerate -m "make target_class nullable"
```

Leer siempre el archivo generado en `versions/` antes de aplicarlo. El
autogenerado detecta tablas y columnas, pero se le escapan los cambios de tipo,
los renombrados (los ve como borrar más crear) y toda migración de datos.

## 7. Ejemplo

La migración `022 -> 023` volvió `target_class` NULLABLE. Respaldo, `current`
confirmó `022`, el `upgrade head` reportó `Running upgrade 022 -> 023`, y la
verificación mostró la base en `023` con la columna nullable y las 29 filas
intactas.
