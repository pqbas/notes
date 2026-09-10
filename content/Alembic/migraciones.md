---
title: Aplicar migraciones de base de datos con Alembic
tags:
  - alembic
  - database
  - python
---

Alembic versiona el esquema de una base de datos como una cadena de revisiones,
donde cada una sabe subir (`upgrade`) y bajar (`downgrade`), y la propia base
guarda en la tabla `alembic_version` en qué revisión está parada. Los pasos
siguientes aplican una migración pendiente sin perder datos y dejando forma de
volver atrás si algo sale mal.

## 1. Respaldo antes de tocar nada

El respaldo se hace antes de cualquier comando de escritura, porque un
`upgrade` que borra o transforma columnas no siempre se revierte limpio con el
`downgrade`, y con SQLite basta copiar el archivo.

```bash
cp data/robot/robot.db "data/robot/robot.db.bak-pre023-$(date +%Y%m%d-%H%M%S)"
```

El sufijo con la revisión de destino y la marca de tiempo evita pisar respaldos
anteriores y deja claro desde qué punto se puede restaurar.

En PostgreSQL el equivalente es un volcado lógico del esquema y los datos.

```bash
pg_dump -Fc -f backup-pre023.dump "$DATABASE_URL"
```

## 2. Revisión en la que está la base

La revisión actual se consulta antes de migrar, ya que es la que confirma desde
dónde se parte y si la base está realmente al día.

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini current
```

La salida es el identificador de la revisión, por ejemplo `022`. Si sale vacía,
la tabla `alembic_version` no existe todavía y la base nunca fue migrada.

| Comando                  | Qué muestra                                           |
| ------------------------ | ----------------------------------------------------- |
| `alembic current`        | la revisión en la que está la base ahora mismo        |
| `alembic heads`          | la última revisión disponible en el código            |
| `alembic history`        | la cadena completa de revisiones, de la más nueva     |
| `alembic history -r022:` | solo lo que falta aplicar desde `022` en adelante     |

Cuando `current` y `heads` coinciden no hay nada pendiente, y si `heads`
devuelve más de una línea hay ramas sin fusionar, que se resuelven con
`alembic merge` antes de migrar.

## 3. Aplicación de la migración

El `upgrade head` avanza la base hasta la última revisión, aplicando en orden
todas las intermedias que falten.

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini upgrade head
```

La salida nombra cada salto que ejecuta, por ejemplo `Running upgrade 022 ->
023`, y ese es el registro de qué se aplicó realmente.

Las tres piezas del comando importan porque Alembic no las adivina.

| Elemento                     | Qué aporta                                              |
| ---------------------------- | ------------------------------------------------------- |
| `ENV_FILE=.env.robot`        | elige a qué base apunta, si el proyecto lee la URL de ahí |
| `-c src/back/alembic.ini`    | ubica el `alembic.ini` cuando no está en la raíz         |
| `uv run`                     | ejecuta dentro del entorno del proyecto                  |

El `ENV_FILE` explícito hace falta cuando el atajo del `Makefile` apunta a otro
entorno. En este proyecto `make db-migrate` tiene `.env.server` fijo en el
target, así que en el robot hay que invocar Alembic directo con `.env.robot`,
porque de lo contrario la migración se aplicaría sobre la base equivocada.

Para avanzar de a un paso, en vez de saltar hasta el final, se usa `+1`, que
conviene cuando hay varias revisiones pendientes y se quiere revisar el estado
entre una y otra.

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini upgrade +1
```

## 4. Verificación posterior

La comprobación tiene tres niveles, y separarlos indica si la migración quedó a
medias.

```bash
# la base quedó en la revisión esperada
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini current

# el esquema cambió como decía la revisión
sqlite3 data/robot/robot.db ".schema sessions"

# los datos siguen ahí
sqlite3 data/robot/robot.db "SELECT COUNT(*) FROM sessions;"
```

Verificar el conteo de filas antes y después es lo que distingue una migración
que solo alteró el esquema de una que además tocó datos sin querer.

## 5. Volver atrás

El `downgrade` deshace revisiones, y `-1` retrocede exactamente una.

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini downgrade -1
```

La reversión solo funciona si la revisión implementó su función `downgrade`, ya
que muchas la dejan vacía o con `pass`, en cuyo caso el comando no falla pero
tampoco deshace nada. Por eso el respaldo del paso 1 es la garantía real y el
`downgrade` solo la vía cómoda.

Restaurar desde el respaldo es el camino cuando el `downgrade` no existe o dejó
la base inconsistente.

```bash
cp data/robot/robot.db.bak-pre023-20260910-025000 data/robot/robot.db
```

## 6. Crear una revisión nueva

La revisión se genera comparando los modelos de SQLAlchemy contra el esquema
real de la base, y `-m` describe el cambio en el nombre del archivo.

```bash
ENV_FILE=.env.robot uv run alembic -c src/back/alembic.ini revision \
  --autogenerate -m "make target_class nullable"
```

El archivo generado bajo `versions/` siempre se lee antes de aplicarlo, porque
el autogenerado detecta columnas y tablas pero se le escapan los cambios de
tipo, los renombrados, que interpreta como borrar más crear, y todo lo que sea
migración de datos.

## 7. Ejemplo de ejecución

La migración `022 -> 023` sobre la base del robot volvió `target_class`
NULLABLE. El respaldo se tomó primero, `current` confirmó `022`, el `upgrade
head` reportó `Running upgrade 022 -> 023`, y la verificación posterior mostró
la base en `023` con la columna ya nullable y las 29 filas de la tabla
intactas.
