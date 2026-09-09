---
title: Modelo de administración en Snowflake
tags:
  - snowflake
  - data
---

## 1. Qué es?

Snowflake es una plataforma que permite:

1. Almacenar datos
2. Procesar datos
3. Analizar datos

## 2. Cómo funciona?

En Snowflake la data y la computo viven separadas, a diferencia de otras bases
de datos como PostgreSQL

Snowflake se separa en tres capas:

- Servicios: Los proporciona snowflake y permiten operar sobre tus datos
  - autenticación
  - control de acceso
  - etc

- Computo (Warehouse): Se encarga de ejecutar las consultas
  - EC3 en Amazon

- Almacenamiento: Se encarga de almacenar los datos
  - S3 en AWS
  - Blob Storage en Azure
  - GCS en Google

## 3. Objetos de snowflake

En snowflake todo es un objeto (usuario, tabla, rol, agente cortex, warehouse),
los cuales poseen las siguientes propiedades:

| Propiedad            | Qué es                                                              |
| -------------------- | ------------------------------------------------------------------- |
| Nombre               | identificador único dentro de su contenedor                         |
| Nombre calificado    | la ruta completa de contenedores que lleva hasta el objeto          |
| Tipo                 | determina qué privilegios acepta y qué opciones admite su `ALTER`   |
| Dueño                | el rol que lo creó, que puede todo sobre él sin recibir privilegios |
| Contenedor           | el objeto que lo contiene, que es la cuenta, una base o un esquema  |
| Propiedades del tipo | los atributos que solo existen para ese tipo de objeto              |

A continuación se describen algunos objetos y la jerarquía que Snowflake les dá:

| Objeto                | Qué es                                                                    |
| --------------------- | ------------------------------------------------------------------------- |
| Warehouse             | cómputo que ejecuta las consultas, sin contener data                      |
| Rol                   | el recipiente con nombre donde se acumulan los privilegios                |
| Usuario               | la identidad con la que alguien se conecta                                |
| Security Integration  | la conexión entre Snowflake y un proveedor de identidad externo           |
| Base de datos         | el contenedor de esquemas                                                 |
| Esquema               | el contenedor de los objetos de trabajo                                   |
| Tabla                 | la estructura que guarda las filas                                        |
| Vista                 | una consulta guardada que se comporta como tabla                          |
| Vista semántica       | el vocabulario de negocio que Cortex Analyst usa para traducir a SQL      |
| Cortex Search Service | el índice semántico sobre una columna de texto                            |
| Agente                | el orquestador que decide qué herramienta atiende cada pregunta           |
| Procedimiento         | código que se ejecuta dentro de Snowflake y expone el agente hacia afuera |

A continuación se muestra la jerarquia presente en Snowflake (cada elemento es
un objeto):

```
Cuenta
├── Warehouse                       el cómputo, se factura por segundo encendido
├── Rol                             el conjunto de privilegios
├── Usuario                         quien se conecta
├── Security Integration            la conexión con Entra ID para OAuth
└── Base de datos
    └── Esquema
        ├── Tabla y vista
        ├── Vista semántica          configuración de Cortex Analyst
        ├── Cortex Search Service    configuración de Cortex Search
        ├── Agente                   orquestador que usa las dos anteriores
        └── Procedimiento almacenado
```

> [!WARNING] Los privilegios otorgados no son una propiedad del objeto sino una
> relación entre el objeto y un rol

Sobre los objetos se ejecutan las siguientes operaciones:

| Operación                         | Objetos a los que aplica                                                                              |
| --------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `CREATE`, `ALTER`, `DROP`, `SHOW` | todos                                                                                                 |
| `GRANT` y `REVOKE`                | todos menos el usuario, sobre el que solo se transfiere la propiedad                                  |
| `DESCRIBE`                        | los que tienen estructura interna: tabla, vista, vista semántica, procedimiento, integración, usuario |
| `UNDROP`                          | base de datos, esquema y tabla, que son los que tienen Time Travel                                    |
| `CLONE`                           | base de datos, esquema y tabla                                                                        |
| `USE`                             | lo que define el contexto de una sesión: rol, warehouse, base y esquema                               |

## 4. Creación de bases de datos, esquema y tabla

Los tres objetos se crean en una sola secuencia, donde el orden importa porque
cada sentencia necesita que exista el contenedor de la anterior y porque el
`USE SCHEMA` solo puede apuntar a un esquema ya creado.

```sql
--- Declaramos el uso del rol SYSADMIN
--- que trae el privilegio de "CREATE DATABASE"
USE ROLE      SYSADMIN;

--- Usamos el warehouse por defecto
USE WAREHOUSE COMPUTE_WH;


--- Creamos el database y schema
CREATE DATABASE IF NOT EXISTS my_db;
CREATE SCHEMA   IF NOT EXISTS my_db.my_sc;

USE SCHEMA my_db.my_sc;


--- Creamos la tabla "MI_TABLA"
CREATE TABLE IF NOT EXISTS MI_TABLA (
  id     NUMBER(10)   NOT NULL COMMENT 'identificador del registro',
  nombre VARCHAR(100) NOT NULL COMMENT 'nombre del registro',
  monto  NUMBER(18,2)          COMMENT 'importe asociado',
  fecha  DATE                  COMMENT 'fecha del registro'
);
```

## 5. Poblado de la tabla con datos aleatorios

Las filas de prueba se generan dentro de Snowflake con `GENERATOR`, sin archivo
de por medio, donde el `INSERT` toma como origen un `SELECT` que fabrica cada
columna en lugar de leerla de otra tabla.

```sql
--- Insertamos 50 filas con datos aleatorios en MI_TABLA
INSERT INTO my_db.my_sc.MI_TABLA (id, nombre, monto, fecha)
SELECT
  SEQ4() + 1                                              AS id,
  'CLIENTE_' || LPAD(SEQ4() + 1, 3, '0')                  AS nombre,
  ROUND(UNIFORM(100, 5000, RANDOM()), 2)                  AS monto,
  DATEADD(day, -UNIFORM(0, 180, RANDOM()), CURRENT_DATE()) AS fecha
FROM TABLE(GENERATOR(ROWCOUNT => 50));
```

Cada función cumple un rol dentro de la generación.

| Función                            | Qué aporta                                                    |
| ---------------------------------- | ------------------------------------------------------------- |
| `GENERATOR(ROWCOUNT => 50)`        | produce las 50 filas sobre las que se evalúa el `SELECT`      |
| `SEQ4()`                           | numera las filas desde 0, por lo que sirve de correlativo     |
| `LPAD(valor, 3, '0')`              | rellena con ceros a la izquierda para armar el código         |
| `UNIFORM(min, max, RANDOM())`      | sortea un valor entero dentro del rango indicado              |
| `DATEADD(day, -n, CURRENT_DATE())` | reparte las fechas dentro de los últimos 180 días             |

El `INSERT` se puede ejecutar varias veces, por lo que repetirlo duplica los
`id`, dado que la tabla no declara clave primaria que lo impida.

## 6. Consulta de la tabla y de su estructura

Las filas se obtienen con `SELECT`, que es la sentencia que devuelve el
contenido, mientras que la estructura del objeto se consulta con `SHOW` y
`DESCRIBE`, que no leen filas.

```sql
--- Traemos todas las filas de la tabla
SELECT * FROM my_db.my_sc.MI_TABLA;

--- Traemos solo una muestra, para no leer toda la tabla
SELECT * FROM my_db.my_sc.MI_TABLA LIMIT 10;

--- Con el esquema ya activo en la sesión basta el nombre corto
SELECT * FROM MI_TABLA;
```

El nombre corto funciona solo mientras el `USE SCHEMA my_db.my_sc` siga activo
en la sesión, por lo que en un script que se ejecuta completo conviene el nombre
calificado.

```sql
--- Listamos qué tablas existen en el esquema, con sus filas y su tamaño
SHOW TABLES IN SCHEMA my_db.my_sc;

--- Vemos las columnas, sus tipos y los COMMENT declarados al crearla
DESCRIBE TABLE my_db.my_sc.MI_TABLA;
```

En Snowsight las dos vistas están en **Data → Databases → my_db → my_sc →
MI_TABLA**, donde la pestaña *Columns* muestra la estructura y *Data Preview*
las primeras filas.
