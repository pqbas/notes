---
title: Crear una API desde cero
tags:
  - backend
  - api
  - fastapi
  - python
  - tutorial
  - diseño
---

La presente nota es un tutorial sencillo de cómo crear una API para una lista
TODO, con las funcionalidades CRUD completas para una tarea (`task`).

El énfasis está en el diseño agregando características, no en una implementación
prolija ni en proponer una solución escalable. El recorrido va de los requisitos
al modelo de datos, y del modelo a una API que lo soporte; el código es el
mínimo necesario para que esa cadena se vea.

El contenido se distribuye entonces de la siguiente manera:

1. [Requisitos funcionales](#1-requisitos-funcionales)
2. [Modelo de datos y contrato de endpoints](#2-modelo-de-datos-y-contrato-de-endpoints)
3. [Proyecto, dependencias y organización de carpetas](#3-proyecto-dependencias-y-organización-de-carpetas)
4. [Base del sistema con la aplicación y el router vacío](#4-base-del-sistema-con-la-aplicación-y-el-router-vacío)
5. [Funcionalidad de creación de tareas](#5-funcionalidad-de-creación-de-tareas)
6. [Funcionalidad de lectura de una tarea](#6-funcionalidad-de-lectura-de-una-tarea)
7. [Funcionalidad de listado con filtro y paginación](#7-funcionalidad-de-listado-con-filtro-y-paginación)
8. [Funcionalidad de actualización parcial](#8-funcionalidad-de-actualización-parcial)
9. [Funcionalidad de borrado](#9-funcionalidad-de-borrado)
10. [Documentación automática con OpenAPI](#10-documentación-automática-con-openapi)
11. [Capas pendientes para una API en producción](#11-capas-pendientes-para-una-api-en-producción)

## 1. Requisitos funcionales

A continuación se propone un conjunto mínimo de características que debe
satisfacer la API, nótese que se busca describir en lenguaje natural las
propiedades y operaciones a realizar sobre las tareas:

1. Crear una tarea con un nombre y, opcionalmente, descripción y fecha límite.
2. Obtener una tarea por su identificador.
3. Listar las tareas, con paginación y filtro por estado.
4. Actualizar una tarea, cambiando solo algunos de sus campos.
5. Borrar una tarea.

## 2. Modelo de datos y contrato de endpoints

Con los requisitos definidos, toca modelar el objeto principal: la `task`. De
los requisitos salen `name`, `description`, `status` y `due_date`. Sin embargo
como dos tareas pueden tener el mismo nombre, hace falta un `id` para poder
señalar una en concreto. Por lo que el modelo queda:

| Campo         | Tipo                     | Descripción                          |
| ------------- | ------------------------ | ------------------------------------ |
| `id`          | `str`                    | Identificador, lo asigna el servidor |
| `name`        | `str`                    | Nombre de la tarea, obligatorio      |
| `description` | `str` o nulo             | Texto libre, opcional                |
| `status`      | `todo`, `doing` o `done` | Estado actual                        |
| `due_date`    | `datetime` o nulo        | Fecha límite, opcional               |
| `created_at`  | `datetime`               | Cuándo se creó, lo pone el servidor  |
| `updated_at`  | `datetime` o nulo        | Última modificación, nulo si nunca   |

Con el modelo definido, cada acción del flujo se traduce a un endpoint según la
convención REST, donde el path nombra el recurso en plural y el verbo HTTP
nombra la acción.

| Acción     | Endpoint             | Éxito | Errores      |
| ---------- | -------------------- | ----- | ------------ |
| Crear      | `POST /tasks`        | `201` | `422`        |
| Leer una   | `GET /tasks/{id}`    | `200` | `404`        |
| Listar     | `GET /tasks`         | `200` | `422`        |
| Actualizar | `PATCH /tasks/{id}`  | `200` | `404`, `422` |
| Borrar     | `DELETE /tasks/{id}` | `204` | `404`        |

Los cinco status codes de esta tabla cubren la mayoría de los casos de cualquier
CRUD, por lo que conviene tener claro cuándo corresponde cada uno antes de
escribirlos.

| Código | Cuándo se usa                                              |
| ------ | ---------------------------------------------------------- |
| `200`  | La operación salió bien y la respuesta lleva el recurso    |
| `201`  | Se creó un recurso nuevo y la respuesta lo devuelve        |
| `204`  | La operación salió bien y no hay nada que devolver         |
| `404`  | El recurso pedido no existe                                |
| `422`  | El body llegó bien formado pero es inválido para el modelo |

Una decisión del contrato que conviene fijar en esta etapa es que el `id`, el
`created_at` y el `status` inicial los asigna el servidor y no el cliente,
porque si el cliente pudiera enviarlos, dos clientes podrían elegir el mismo
`id`.

## 3. Proyecto, dependencias y organización de carpetas

El proyecto se crea con `uv`, que resuelve las dependencias y administra el
entorno virtual sin pasos manuales.

```bash
# Creando el proyecto y entrando a la carpeta
uv init tasks-api
cd tasks-api

# Agregando las dependencias
uv add fastapi "uvicorn[standard]"
```

La aplicación necesita dos paquetes, cuyo aporte resume la tabla que sigue al
comando de instalación.

| Paquete   | Descripcion                                                  |
| --------- | ------------------------------------------------------------ |
| `fastapi` | Permite crear la API (qué hace una vez llega el diccionario) |
| `uvicorn` | Servidor ASGI (traduce HTTP a diccionario)                   |

Una vez creado el proyecto tendrá que eliminar y crear archivos/carpetas de
acuerdo a la siguiente estructura:

```text
tasks-api/
├── .python-version
├── README.md
├── pyproject.toml
├── uv.lock
└── app/
    ├── __init__.py
    ├── main.py
    ├── schemas.py
    ├── store.py
    └── routers/
        ├── __init__.py
        └── tasks.py
```

La siguiente tabla indica la responsabilidad de cada archivo:

| Archivo                | Responsabilidad                                         | Secciones |
| ---------------------- | ------------------------------------------------------- | --------- |
| `app/main.py`          | Crea la instancia de `FastAPI` y registra los routers   | 4         |
| `app/routers/tasks.py` | Los endpoints del recurso `/tasks`                      | 4 a 9     |
| `app/schemas.py`       | Los modelos de entrada y salida                         | 5, 7 y 8  |
| `app/store.py`         | El almacén en memoria                                   | 5         |
| `__init__.py`          | Marca la carpeta como paquete de Python, puede ir vacío | 3         |

## 4. Base del sistema con la aplicación y el router vacío

| Archivo                | Qué se agrega                                          |
| ---------------------- | ------------------------------------------------------ |
| `app/routers/tasks.py` | El objeto `router`, todavía sin endpoints              |
| `app/main.py`          | La instancia `app`, el registro del router y `/health` |

### 4.1 Router en `app/routers/tasks.py`

Un `APIRouter` agrupa las rutas de un recurso bajo un prefijo común, de modo que
dentro del router los paths se escriben relativos a `/tasks`, y
`@router.post("")` atiende `POST /tasks` mientras que
`@router.get("/{task_id}")` atiende `GET /tasks/{task_id}`.

```python
# app/routers/tasks.py
from fastapi import APIRouter

router = APIRouter(prefix="/tasks", tags=["tasks"])
```

### 4.2 Aplicación en `app/main.py`

En el archivo `app/main.py` crea la instancia de `FastAPI` y la conecta con el
router mediante `include_router`. Un router que nadie registra no atiende
ninguna petición.

Además crea la ruta `/health`, que es una verificación de que el servidor está
vivo, que responde `{"status": "ok"}` sin tocar datos ni recursos, y por eso se
queda en `main.py` en lugar de ir a un router.

```python
# app/main.py
from fastapi import FastAPI

from app.routers import tasks

app = FastAPI(title="Tasks API")
app.include_router(tasks.router)


@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

La ruta `health` se usa siempre por balanceadores de carga y orquestadores como
Kubernetes para decidir si una instancia recibe tráfico, así que casi toda API
en producción tiene una.

### 4.3 Servidor en marcha

Levantando el servidor en modo desarrollo, la opción `--reload` reinicia el
proceso cada vez que cambia un archivo, por lo que el servidor puede quedar
corriendo mientras se agregan las funcionalidades.

```bash
uv run uvicorn app.main:app --reload
```

```text
INFO:     Will watch for changes in these directories: ['/home/pqbas/tasks-api']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [109468] using WatchFiles
INFO:     Started server process [109485]
INFO:     Application startup complete.
```

Para validar que el servidor está encendido, y respondiendo, puede hacer
consultas al endpoint y puerto correspondiente:

```bash
curl -i http://127.0.0.1:8000/health
```

```text
HTTP/1.1 200 OK
content-type: application/json

{"status":"ok"}
```

## 5. Funcionalidad de creación de tareas

| Archivo                | Qué se agrega                                          |
| ---------------------- | ------------------------------------------------------ |
| `app/schemas.py`       | `Status`, `TaskCreate` y `Task`                        |
| `app/store.py`         | El diccionario `db`                                    |
| `app/routers/tasks.py` | Los imports de la creación y el endpoint `create_task` |

Agregue el siguiente código en `app/schemas.py`.

```python
# app/schemas.py
from datetime import datetime
from typing import Literal

from pydantic import BaseModel, Field

Status = Literal["todo", "doing", "done"]


class TaskCreate(BaseModel):
    name: str = Field(min_length=1)
    description: str | None = None
    due_date: datetime | None = None


class Task(BaseModel):
    id: str
    name: str
    description: str | None
    status: Status
    created_at: datetime
    updated_at: datetime | None
    due_date: datetime | None
```

Agregue el siguiente código en `app/store.py`.

```python
# app/store.py
from app.schemas import Task

db: dict[str, Task] = {}
```

Agregue el siguiente código en `app/routers/tasks.py`.

```python
# app/routers/tasks.py
from datetime import datetime, timezone
from uuid import uuid4

from fastapi import APIRouter

from app.schemas import Task, TaskCreate
from app.store import db

router = APIRouter(prefix="/tasks", tags=["tasks"])


@router.post("", status_code=201)
def create_task(body: TaskCreate) -> Task:
    task = Task(
        id=str(uuid4()),
        name=body.name,
        description=body.description,
        status="todo",
        created_at=datetime.now(timezone.utc),
        updated_at=None,
        due_date=body.due_date,
    )
    db[task.id] = task
    return task
```

Ejecute los siguientes comandos para validar que la creación responde `201` con
la tarea completa, y `422` cuando falta el nombre.

```bash
curl -i -X POST http://127.0.0.1:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"name": "Comprar leche"}'
```

```http
HTTP/1.1 201 Created
content-type: application/json

{
  "id": "a211f5b3-a524-4b20-8d79-c305cab17ce7",
  "name": "Comprar leche",
  "description": null,
  "status": "todo",
  "created_at": "2026-09-22T18:08:32.632592Z",
  "updated_at": null,
  "due_date": null
}
```

```bash
curl -X POST http://127.0.0.1:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"name": "Escribir la nota", "description": "Tutorial de API"}'
```

```json
{
  "id": "10a79592-0c2f-4ab6-8c0d-f1d79d5173e3",
  "name": "Escribir la nota",
  "description": "Tutorial de API",
  "status": "todo",
  "created_at": "2026-09-22T18:08:32.637555Z",
  "updated_at": null,
  "due_date": null
}
```

```bash
curl -X POST http://127.0.0.1:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"description": "sin nombre"}'
```

```json
{
  "detail": [
    {
      "type": "missing",
      "loc": ["body", "name"],
      "msg": "Field required",
      "input": { "description": "sin nombre" }
    }
  ]
}
```

## 6. Funcionalidad de lectura de una tarea

| Archivo                | Qué se agrega                                          |
| ---------------------- | ------------------------------------------------------ |
| `app/schemas.py`       | Nada, reutiliza `Task`                                 |
| `app/routers/tasks.py` | El import de `HTTPException` y el endpoint `read_task` |

Agregue el siguiente código en `app/routers/tasks.py`.

```python
# app/routers/tasks.py
from fastapi import APIRouter, HTTPException

...

@router.get("/{task_id}")
def read_task(task_id: str) -> Task:
    task = db.get(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="task not found")
    return task
```

Ejecute los siguientes comandos para validar que la lectura devuelve la tarea
con `200`, y `404` cuando el id no existe.

```bash
curl -i http://127.0.0.1:8000/tasks/10a79592-0c2f-4ab6-8c0d-f1d79d5173e3
```

```http
HTTP/1.1 200 OK
content-type: application/json

{
  "id": "10a79592-0c2f-4ab6-8c0d-f1d79d5173e3",
  "name": "Escribir la nota",
  "description": "Tutorial de API",
  "status": "todo",
  "created_at": "2026-09-22T18:08:32.637555Z",
  "updated_at": null,
  "due_date": null
}
```

```bash
curl -i http://127.0.0.1:8000/tasks/no-existe
```

```text
HTTP/1.1 404 Not Found
content-type: application/json

{"detail":"task not found"}
```

## 7. Funcionalidad de listado con filtro y paginación

| Archivo                | Qué se agrega                                                   |
| ---------------------- | --------------------------------------------------------------- |
| `app/schemas.py`       | `TaskPage`                                                      |
| `app/routers/tasks.py` | Los imports de `Status` y `TaskPage` y el endpoint `list_tasks` |

Agregue el siguiente código al final de `app/schemas.py`.

```python
# app/schemas.py, al final del archivo
class TaskPage(BaseModel):
    items: list[Task]
    total: int
```

Agregue el siguiente código en `app/routers/tasks.py`.

```python
# app/routers/tasks.py
from app.schemas import Status, Task, TaskCreate, TaskPage

...

@router.get("")
def list_tasks(
    status: Status | None = None,
    limit: int = 20,
    offset: int = 0,
) -> TaskPage:
    items = list(db.values())
    if status is not None:
        items = [task for task in items if task.status == status]
    return TaskPage(items=items[offset : offset + limit], total=len(items))
```

| Parámetro | Por defecto | Efecto                                 |
| --------- | ----------- | -------------------------------------- |
| `status`  | sin filtro  | Devuelve solo las tareas en ese estado |
| `limit`   | `20`        | Cuántos elementos trae la página       |
| `offset`  | `0`         | Desde qué posición empieza la página   |

Ejecute los siguientes comandos para validar que el listado pagina con `limit` y
rechaza con `422` un estado fuera del conjunto.

```bash
curl "http://127.0.0.1:8000/tasks?limit=1"
```

```json
{
  "items": [
    {
      "id": "a211f5b3-a524-4b20-8d79-c305cab17ce7",
      "name": "Comprar leche",
      "description": null,
      "status": "todo",
      "created_at": "2026-09-22T18:08:32.632592Z",
      "updated_at": null,
      "due_date": null
    }
  ],
  "total": 2
}
```

```bash
curl "http://127.0.0.1:8000/tasks?status=pendiente"
```

```json
{
  "detail": [
    {
      "type": "literal_error",
      "loc": ["query", "status"],
      "msg": "Input should be 'todo', 'doing' or 'done'",
      "input": "pendiente",
      "ctx": { "expected": "'todo', 'doing' or 'done'" }
    }
  ]
}
```

## 8. Funcionalidad de actualización parcial

| Archivo                | Qué se agrega                                         |
| ---------------------- | ----------------------------------------------------- |
| `app/schemas.py`       | `TaskUpdate`                                          |
| `app/routers/tasks.py` | El import de `TaskUpdate` y el endpoint `update_task` |

Agregue el siguiente código al final de `app/schemas.py`.

```python
# app/schemas.py, al final del archivo
class TaskUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=1)
    description: str | None = None
    status: Status | None = None
    due_date: datetime | None = None
```

Agregue el siguiente código en `app/routers/tasks.py`.

```python
# app/routers/tasks.py
from app.schemas import Status, Task, TaskCreate, TaskPage, TaskUpdate

...

@router.patch("/{task_id}")
def update_task(task_id: str, body: TaskUpdate) -> Task:
    task = db.get(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="task not found")

    changes = body.model_dump(exclude_unset=True)
    changes["updated_at"] = datetime.now(timezone.utc)

    updated = task.model_copy(update=changes)
    db[task_id] = updated
    return updated
```

| Body enviado            | Qué pasa con `description`    |
| ----------------------- | ----------------------------- |
| `{"status": "done"}`    | Se conserva el valor anterior |
| `{"description": null}` | Se borra y queda en nulo      |

Ejecute los siguientes comandos para validar que la actualización parcial cambia
solo los campos enviados, y rechaza con `422` un estado inválido.

```bash
curl -X PATCH http://127.0.0.1:8000/tasks/10a79592-0c2f-4ab6-8c0d-f1d79d5173e3 \
  -H "Content-Type: application/json" \
  -d '{"status": "doing"}'
```

```json
{
  "id": "10a79592-0c2f-4ab6-8c0d-f1d79d5173e3",
  "name": "Escribir la nota",
  "description": "Tutorial de API",
  "status": "doing",
  "created_at": "2026-09-22T18:08:32.637555Z",
  "updated_at": "2026-09-22T18:08:32.684988Z",
  "due_date": null
}
```

```bash
curl -X PATCH http://127.0.0.1:8000/tasks/10a79592-0c2f-4ab6-8c0d-f1d79d5173e3 \
  -H "Content-Type: application/json" \
  -d '{"status": "terminado"}'
```

```json
{
  "detail": [
    {
      "type": "literal_error",
      "loc": ["body", "status"],
      "msg": "Input should be 'todo', 'doing' or 'done'",
      "input": "terminado",
      "ctx": { "expected": "'todo', 'doing' or 'done'" }
    }
  ]
}
```

## 9. Funcionalidad de borrado

| Archivo                | Qué se agrega             |
| ---------------------- | ------------------------- |
| `app/routers/tasks.py` | El endpoint `delete_task` |

Agregue el siguiente código al final de `app/routers/tasks.py`.

```python
# app/routers/tasks.py, al final del archivo
@router.delete("/{task_id}", status_code=204)
def delete_task(task_id: str) -> None:
    if task_id not in db:
        raise HTTPException(status_code=404, detail="task not found")
    del db[task_id]
```

Ejecute los siguientes comandos para validar que el borrado responde `204`, y
`404` al repetirlo sobre el mismo id.

```bash
curl -i -X DELETE http://127.0.0.1:8000/tasks/10a79592-0c2f-4ab6-8c0d-f1d79d5173e3
```

```text
HTTP/1.1 204 No Content
```

```bash
curl -i -X DELETE http://127.0.0.1:8000/tasks/10a79592-0c2f-4ab6-8c0d-f1d79d5173e3
```

```text
HTTP/1.1 404 Not Found
content-type: application/json

{"detail":"task not found"}
```

## 10. Documentación automática con OpenAPI

| Ruta            | Qué contiene                                    |
| --------------- | ----------------------------------------------- |
| `/docs`         | Interfaz web para leer y probar los endpoints   |
| `/openapi.json` | El esquema OpenAPI crudo, para generar clientes |

## 11. Capas pendientes para una API en producción
