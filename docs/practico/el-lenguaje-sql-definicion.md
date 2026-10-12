# El lenguaje SQL: Definición

## Objetivos

| # | Objetivo |
|---|---|
| 1 | Presentar la sintaxis del lenguaje SQL correspondiente al Lenguaje de Definición de Datos (DDL). |
| 2 | Distinguir el DDL del DML (tema anterior) y situar cada instrucción SQL en su categoría. |
| 3 | Comprender la normalización de tablas (1FN, 2FN, 3FN) como paso previo al diseño de una base de datos. |
| 4 | Ver ejemplos prácticos de creación, modificación y eliminación de estructuras (bases de datos, tablas, restricciones, índices y vistas). |
| 5 | Reutilizar la base de datos ACADEMIA como esquema de referencia, para dar continuidad al tema de manipulación de datos. |

!!! note "Sobre los datos de este documento"
    Los ejemplos de este tema parten de la misma base de datos ACADEMIA usada en "El lenguaje SQL: Manipulación" (ver `crear_bd_academia.sql`). Las instrucciones DDL de este documento están recogidas en el script `ddl_academia.sql`, pensado para ejecutarse **después** de `crear_bd_academia.sql`.

    Algunos ejemplos modifican la estructura de ACADEMIA con fines didácticos (añaden y eliminan columnas, restricciones, índices...). Si después se quiere volver a usar `consultas_teoria.sql`, hay que volver a ejecutar `crear_bd_academia.sql` para dejar la base de datos en su estado original.

---

## DDL frente a DML

!!! tip "Idea clave"
    El lenguaje SQL se divide en dos grandes bloques según lo que manipulan:

    - **DDL** (*Data Definition Language*): define y modifica la **estructura** de la base de datos (bases de datos, tablas, columnas, restricciones, índices, vistas). Instrucciones típicas: `CREATE`, `ALTER`, `DROP`, `TRUNCATE`.
    - **DML** (*Data Manipulation Language*): manipula los **datos** que hay dentro de esas estructuras. Instrucciones típicas: `SELECT`, `INSERT`, `UPDATE`, `DELETE` (ya vistas en el tema anterior).

| Instrucción | Categoría | Qué hace |
|---|---|---|
| `CREATE DATABASE` / `CREATE TABLE` | DDL | Crea una base de datos o una tabla nueva |
| `ALTER TABLE` | DDL | Modifica la estructura de una tabla ya existente |
| `DROP DATABASE` / `DROP TABLE` | DDL | Elimina una base de datos o una tabla completa (estructura y datos) |
| `TRUNCATE TABLE` | DDL | Vacía una tabla de todas sus filas, conservando su estructura |
| `CREATE INDEX` / `DROP INDEX` | DDL | Crea o elimina un índice sobre una o varias columnas |
| `CREATE VIEW` / `DROP VIEW` | DDL | Crea o elimina una vista (una consulta guardada con nombre) |
| `SELECT`, `INSERT`, `UPDATE`, `DELETE` | DML | Consultan o modifican los datos, sin tocar la estructura |

Una diferencia práctica importante: en MySQL/MariaDB, la mayoría de las instrucciones DDL hacen un **commit implícito**: no se pueden deshacer con `ROLLBACK` dentro de una transacción, a diferencia de las instrucciones DML.

---

## Creación de bases de datos

!!! tip "Idea clave"
    Antes de crear tablas hace falta una base de datos donde alojarlas. `CREATE DATABASE` la crea (vacía); `USE` selecciona la base de datos activa para las siguientes instrucciones; `DROP DATABASE` la elimina por completo, con todo su contenido.

```sql
CREATE DATABASE IF NOT EXISTS ddl_demo;
DROP DATABASE ddl_demo;
```

`IF NOT EXISTS` evita un error si la base de datos ya existiera; es opcional, pero conviene usarlo en scripts que se puedan ejecutar más de una vez. `DROP DATABASE` no admite deshacerse: elimina físicamente todos los datos y estructuras que contenía.

---

## Normalización de tablas

!!! tip "Idea clave"
    Antes de escribir el primer `CREATE TABLE` hay que decidir **qué tablas hacen falta y qué columnas va en cada una**. La normalización es el proceso que lleva de unos datos "en bruto" (por ejemplo, una hoja de cálculo con todo mezclado) a un conjunto de tablas bien diseñadas, sin redundancias ni información mal repartida. Se describe en **formas normales** (FN), cada una más exigente que la anterior.

**Punto de partida:** una hoja de cálculo típica, con todos los datos en una sola tabla desnormalizada:

| Alumno | Nombre completo | Ciclo | Tutor del ciclo | Módulo | Nota |
|---|---|---|---|---|---|
| 1 | Diego Vidal Soler | DAM | Álvaro Pino | Bases de Datos | 8.5 |
| 1 | Diego Vidal Soler | DAM | Álvaro Pino | Programación | 7.0 |
| 2 | Roberto Díaz Luna | DAM | Álvaro Pino | Bases de Datos | 9.0 |

### Primera forma normal (1FN)

!!! tip "Idea clave"
    Una tabla está en 1FN cuando todas sus columnas contienen **valores atómicos** (no divisibles en partes con sentido propio) y no hay grupos de columnas repetidos.

La columna `Nombre completo` mezcla dos datos distintos (nombre y apellidos) en un único valor: no es atómica. Para estar en 1FN hay que separarla:

| Alumno | Nombre | Apellidos | Ciclo | Tutor del ciclo | Módulo | Nota |
|---|---|---|---|---|---|---|
| 1 | Diego | Vidal Soler | DAM | Álvaro Pino | Bases de Datos | 8.5 |

!!! warning "Un error habitual"
    Guardar "Nombre y Apellidos" en una sola columna es, con diferencia, la violación de la 1FN más frecuente en los diseños de quien empieza. El síntoma típico es tener que recurrir a funciones de texto (buscar un espacio, cortar la cadena...) para poder ordenar o filtrar solo por apellido; si eso hace falta, es una señal de que esa columna debería estar partida desde el diseño.

### Segunda forma normal (2FN)

!!! tip "Idea clave"
    Una tabla está en 2FN cuando está en 1FN y, además, **todo atributo no clave depende de la clave completa**, no solo de una parte de ella (esto solo es relevante cuando la clave primaria está formada por varias columnas).

En la tabla anterior, si la clave fuera `(Alumno, Módulo)`, el `Nombre` y los `Apellidos` dependerían solo de `Alumno` (se repiten igual en todas las filas de ese alumno, tenga las notas que tenga), no de la clave completa. Eso es una dependencia parcial, y obliga a separar `Alumno` en su propia tabla:

- **Alumnos** (<u>alumno</u>, nombre, apellidos, ciclo, tutor_del_ciclo)
- **Notas** (<u>alumno, módulo</u>, nota)

### Tercera forma normal (3FN)

!!! tip "Idea clave"
    Una tabla está en 3FN cuando está en 2FN y, además, **ningún atributo no clave depende de otro atributo no clave** (solo de la clave). A esto se le llama **dependencia transitiva**.

En `Alumnos`, `tutor_del_ciclo` no depende del alumno, sino del `ciclo` (todos los alumnos del mismo ciclo comparten el mismo tutor): es una dependencia transitiva (`alumno → ciclo → tutor_del_ciclo`). La solución es sacar `Ciclo` a su propia tabla:

- **Ciclos** (<u>ciclo</u>, tutor)
- **Alumnos** (<u>alumno</u>, nombre, apellidos, ciclo)
- **Notas** (<u>alumno, módulo</u>, nota)

Esta es, precisamente, la estructura de `Ciclo` y `Alumno` que usa la base de datos ACADEMIA de este tema: no es casualidad, sino el resultado de aplicar este mismo razonamiento de normalización.

!!! note "¿Y las formas normales superiores?"
    Existen formas normales más exigentes (FNBC, 4FN, 5FN), pero en la práctica profesional y en el diseño de bases de datos de gestión, llegar a 3FN resuelve la gran mayoría de problemas de redundancia e inconsistencia. Es el nivel que se suele pedir en los enunciados de examen ("normaliza hasta 3FN").

### Resumen: cómo detectar qué forma normal falla

| Si ves esto... | Falla la... |
|---|---|
| Una columna mezcla varios datos (nombre + apellidos, dirección completa sin separar) | 1FN |
| Una columna con listas o varios valores repetidos en la misma celda | 1FN |
| Con clave compuesta, un atributo que solo depende de una parte de la clave | 2FN |
| Un atributo no clave que depende de otro atributo no clave, no de la clave | 3FN |

---

## Creación de tablas: CREATE TABLE

### Tipos de datos más habituales

| Tipo | Uso típico | Ejemplo |
|---|---|---|
| `INT` | Números enteros (identificadores, contadores) | `edad INT` |
| `DECIMAL(p,d)` | Números con decimales exactos (dinero, notas) | `importe DECIMAL(6,2)` |
| `VARCHAR(n)` | Texto de longitud variable, hasta `n` caracteres | `nombre VARCHAR(60)` |
| `CHAR(n)` | Texto de longitud fija (poco usado salvo códigos cortos) | `sexo CHAR(1)` |
| `DATE` | Fechas sin hora (`AAAA-MM-DD`) | `fecha_nacimiento DATE` |
| `DATETIME` / `TIMESTAMP` | Fecha y hora | `fecha_alta DATETIME` |
| `BOOLEAN` (alias de `TINYINT(1)`) | Valores lógicos (0/1) | `activo BOOLEAN` |
| `TEXT` | Texto largo, sin límite práctico de longitud | `observaciones TEXT` |

!!! note "DECIMAL frente a FLOAT/DOUBLE"
    Para dinero o notas académicas se usa `DECIMAL(p,d)` en lugar de `FLOAT`/`DOUBLE`: `DECIMAL` almacena exactamente el número de decimales indicado, sin errores de redondeo binario. Es el tipo usado en ACADEMIA para `Nota.nota` (`DECIMAL(4,2)`) y `Distincion.importe` (`DECIMAL(6,2)`).

### Restricciones de columna

!!! tip "Idea clave"
    Las restricciones (*constraints*) se pueden declarar junto a la columna (restricción **de columna**) o al final de la tabla (restricción **de tabla**, necesaria cuando afecta a varias columnas a la vez, como una clave primaria compuesta).

| Restricción | Efecto |
|---|---|
| `NOT NULL` | La columna no admite valores `NULL` |
| `DEFAULT valor` | Si no se indica un valor al insertar, se usa este por defecto |
| `UNIQUE` | No puede haber dos filas con el mismo valor en esa columna |
| `CHECK (condición)` | Solo se admiten filas que cumplan la condición |
| `AUTO_INCREMENT` | El valor se genera automáticamente, incrementando en cada fila nueva |
| `PRIMARY KEY` | Identifica de forma única cada fila (implica `NOT NULL` + `UNIQUE`) |

**EJEMPLO:** creación de la tabla `Ciclo`, con clave primaria numérica autogenerada y una columna de texto que debe seguir siendo única.

```sql
CREATE TABLE Ciclo (
    idciclo    INT          NOT NULL AUTO_INCREMENT,
    codciclo   VARCHAR(10)  NOT NULL,
    tutor      VARCHAR(60)  NOT NULL,
    PRIMARY KEY (idciclo),
    UNIQUE KEY uq_ciclo_codciclo (codciclo)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_spanish_ci;
```

`idciclo` es la clave primaria (un identificador técnico, numérico y autogenerado); `codciclo` es una clave candidata (`UNIQUE`) que se sigue usando como referencia legible ("DAM", "DAW"...) desde otras tablas.

### Restricciones de tabla y claves ajenas

!!! tip "Idea clave"
    Una **clave ajena** (`FOREIGN KEY`) obliga a que el valor de una columna exista como clave (primaria o única) en otra tabla. Es la forma de implementar en SQL las relaciones del modelo entidad-relación, y garantiza la **integridad referencial**.

**EJEMPLO:** creación de `Alumno`, con clave ajena hacia `Ciclo`.

```sql
CREATE TABLE Alumno (
    nia        INT          NOT NULL,
    nombre     VARCHAR(60)  NOT NULL,
    edad       INT          NULL,
    codciclo   VARCHAR(10)  NOT NULL,
    PRIMARY KEY (nia),
    CONSTRAINT fk_alumno_ciclo FOREIGN KEY (codciclo) REFERENCES Ciclo (codciclo)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_spanish_ci;
```

Darle **nombre explícito** a la restricción (`CONSTRAINT fk_alumno_ciclo ...`) no es obligatorio, pero es muy recomendable: sin nombre, el motor le asigna uno automático, y luego resulta más difícil de identificar si hay que modificarla o eliminarla con `ALTER TABLE`.

**EJEMPLO:** una clave primaria **compuesta** (varias columnas a la vez), en la tabla `Nota`, junto con un `CHECK` que acota el rango de valores válidos.

```sql
CREATE TABLE Nota (
    nia        INT           NOT NULL,
    codmodulo  INT           NOT NULL,
    nota       DECIMAL(4,2)  NOT NULL,
    PRIMARY KEY (nia, codmodulo),
    CONSTRAINT fk_nota_alumno FOREIGN KEY (nia) REFERENCES Alumno (nia),
    CONSTRAINT fk_nota_modulo FOREIGN KEY (codmodulo) REFERENCES Modulo (codmodulo),
    CONSTRAINT ck_nota_rango  CHECK (nota BETWEEN 0 AND 10)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_spanish_ci;
```

`PRIMARY KEY (nia, codmodulo)` significa que la combinación de ambos valores debe ser única: un mismo alumno puede tener notas en varios módulos, y un módulo puede tener notas de varios alumnos, pero no puede haber dos filas con el mismo par `(nia, codmodulo)`.

### ON DELETE / ON UPDATE

!!! tip "Idea clave"
    Cuando se borra o modifica una fila referenciada por una clave ajena, hay que decidir qué pasa con las filas que la referencian. Por defecto (`RESTRICT`), el motor **impide** el borrado si existen referencias; con `ON DELETE CASCADE` las filas dependientes se borran automáticamente; con `ON DELETE SET NULL` la columna de la clave ajena pasa a `NULL` (solo si admite nulos).

| Opción | Efecto al borrar la fila referenciada |
|---|---|
| `RESTRICT` / `NO ACTION` (por defecto) | Impide el borrado mientras existan filas dependientes |
| `CASCADE` | Borra también las filas dependientes |
| `SET NULL` | Pone a `NULL` la clave ajena de las filas dependientes (la columna debe admitir `NULL`) |

**EJEMPLO:** si se define `Obtener_Distincion.nia` con `ON DELETE CASCADE` hacia `Alumno`, al borrar un alumno se borran automáticamente sus distinciones obtenidas:

```sql
ALTER TABLE Obtener_Distincion
    ADD CONSTRAINT fk_obtdist_alumno FOREIGN KEY (nia)
        REFERENCES Alumno (nia) ON DELETE CASCADE;
```

Con esta restricción activa, borrar a Sergio Molina (nia 6, que no tiene notas registradas pero sí una distinción obtenida) elimina en cascada su fila de `Obtener_Distincion`:

```sql
DELETE FROM Alumno WHERE nia = 6;
```

**Comprobado:** antes del borrado había 1 fila en `Obtener_Distincion` con `nia = 6`; después del `DELETE`, 0 filas, y el alumno también ha desaparecido de `Alumno`. Sin `ON DELETE CASCADE`, este mismo `DELETE` habría fallado con un error de restricción de clave ajena (como le ocurre, con la configuración original de ACADEMIA, a cualquier alumno con notas registradas: `Nota.nia` no tiene `CASCADE`, así que primero habría que borrar sus notas).

---

## Modificación de tablas: ALTER TABLE

!!! tip "Idea clave"
    `ALTER TABLE` permite modificar una tabla ya creada, sin tener que borrarla y volver a crearla (lo que además borraría sus datos). Se pueden añadir, modificar o eliminar columnas y restricciones, y renombrar la tabla.

### Añadir columnas

```sql
ALTER TABLE Alumno ADD COLUMN email VARCHAR(100) NULL;
```

La tabla queda así (comprobado con `DESCRIBE Alumno`):

| Field | Type | Null | Key |
|---|---|---|---|
| nia | int(11) | NO | PRI |
| nombre | varchar(60) | NO | |
| edad | int(11) | YES | |
| codciclo | varchar(10) | NO | MUL |
| **email** | **varchar(100)** | **YES** | |

También se puede añadir una restricción en el mismo `ALTER`:

```sql
ALTER TABLE Modulo ADD COLUMN codigo_interno VARCHAR(15) NULL UNIQUE;
```

### Modificar columnas

```sql
ALTER TABLE Alumno MODIFY COLUMN nombre VARCHAR(80) NOT NULL;
```

`MODIFY COLUMN` cambia el tipo o las restricciones de una columna, sin cambiar su nombre. Para cambiar también el nombre, MySQL/MariaDB ofrece `CHANGE COLUMN`:

```sql
ALTER TABLE Modulo CHANGE COLUMN profesor docente VARCHAR(60) NOT NULL;
```

**Comprobado** con `SHOW CREATE TABLE Modulo`: la columna `profesor` pasa a llamarse `docente`, conservando el resto de su definición. Con `CHANGE COLUMN` hay que repetir el tipo de dato completo, aunque no cambie.

### Añadir y eliminar restricciones

```sql
ALTER TABLE Alumno ADD CONSTRAINT ck_alumno_edad CHECK (edad IS NULL OR edad BETWEEN 16 AND 70);
```

**Comprobado:** con esta restricción activa, `INSERT INTO Alumno (nia, nombre, edad, codciclo) VALUES (99, 'Prueba Edad', 5, 'DAM');` falla con el error `CONSTRAINT 'ck_alumno_edad' failed`, porque 5 años queda fuera del rango permitido.

Para eliminarla:

```sql
ALTER TABLE Alumno DROP CONSTRAINT ck_alumno_edad;
```

!!! note "DROP CHECK / DROP CONSTRAINT"
    MySQL 8 admite tanto `DROP CHECK nombre` como `DROP CONSTRAINT nombre` para quitar un `CHECK`. `DROP CONSTRAINT` es la forma más portable, porque también funciona igual en MariaDB.

Para eliminar una clave ajena hace falta nombrarla explícitamente (por eso conviene ponerles nombre al crearlas):

```sql
ALTER TABLE Obtener_Distincion DROP FOREIGN KEY fk_obtdist_alumno;
```

### Eliminar columnas

```sql
ALTER TABLE Modulo DROP COLUMN codigo_interno;
```

Elimina la columna y, con ella, cualquier dato que contuviera: es una operación irreversible salvo que se restaure desde una copia de seguridad.

### Renombrar una tabla

```sql
ALTER TABLE Modulo RENAME TO Asignatura;
```

---

## Eliminación de estructuras: DROP TABLE

!!! tip "Idea clave"
    `DROP TABLE` elimina la tabla por completo: su estructura **y** todos sus datos. A diferencia de `DELETE` (DML), no admite condición `WHERE`: se borra la tabla entera o nada.

```sql
CREATE TABLE Tabla_Prueba (
    id   INT NOT NULL AUTO_INCREMENT,
    dato VARCHAR(30),
    PRIMARY KEY (id)
);

DROP TABLE IF EXISTS Tabla_Prueba;
```

`IF EXISTS` evita un error si la tabla no existiera; es especialmente útil en scripts que se ejecutan varias veces.

!!! warning "Orden de borrado con claves ajenas"
    Si una tabla es referenciada por una clave ajena de otra, no se puede eliminar mientras la segunda exista (o mientras tenga filas que la referencien): hay que borrar primero las tablas "hijas". En ACADEMIA, por ejemplo, no se podría eliminar `Alumno` sin haber eliminado antes `Nota` y `Obtener_Distincion`, que dependen de ella.

---

## TRUNCATE TABLE frente a DELETE

!!! tip "Idea clave"
    `TRUNCATE TABLE` vacía una tabla de todas sus filas de golpe, manteniendo su estructura (columnas, restricciones, índices). A diferencia de `DELETE FROM tabla` (DML), no admite `WHERE` (borra siempre todas las filas), suele ser más rápido, y **reinicia el contador `AUTO_INCREMENT`**.

**EJEMPLO:** se usa una tabla copia para no perder los datos reales de `Nota`.

```sql
CREATE TABLE Nota_Historico LIKE Nota;
INSERT INTO Nota_Historico SELECT * FROM Nota;

TRUNCATE TABLE Nota_Historico;
```

**Comprobado:** antes del `TRUNCATE`, `Nota_Historico` tenía 20 filas (una copia exacta de `Nota`); después, 0 filas. `CREATE TABLE ... LIKE` copia la estructura completa de otra tabla (columnas, tipos, claves) sin copiar sus datos; el `INSERT ... SELECT` posterior es el que copia los datos.

| | `DELETE FROM tabla` | `TRUNCATE TABLE tabla` |
|---|---|---|
| Categoría | DML | DDL |
| Admite `WHERE` | Sí (puede borrar solo algunas filas) | No (borra todas las filas) |
| Reinicia `AUTO_INCREMENT` | No | Sí |
| Se puede deshacer con `ROLLBACK` | Sí (dentro de una transacción) | No (commit implícito) |
| Velocidad | Más lento en tablas grandes | Más rápido |

---

## Otros objetos DDL: índices y vistas

### CREATE INDEX / DROP INDEX

!!! tip "Idea clave"
    Un índice acelera las búsquedas por una columna (a costa de ocupar espacio extra y ralentizar ligeramente las inserciones). Las claves primarias y `UNIQUE` ya crean un índice automáticamente; `CREATE INDEX` permite añadir índices sobre otras columnas que se consultan con frecuencia.

```sql
CREATE INDEX ix_alumno_nombre ON Alumno (nombre);

DROP INDEX ix_alumno_nombre ON Alumno;
```

### CREATE VIEW / DROP VIEW

!!! tip "Idea clave"
    Una vista es una consulta `SELECT` guardada con nombre, que se puede usar como si fuera una tabla. No almacena datos propios: cada vez que se consulta, se vuelve a ejecutar la consulta que la define.

```sql
CREATE VIEW Vista_ModulosHoras AS
    SELECT codmodulo, nombre, horas
    FROM Modulo
    WHERE horas >= 150;

DROP VIEW Vista_ModulosHoras;
```

`SELECT * FROM Vista_ModulosHoras;` se comporta como una consulta normal sobre una tabla, aunque por debajo se está ejecutando el `SELECT` de su definición.

---

## Resumen: verbos DDL

| Verbo | Para qué se usa |
|---|---|
| `CREATE DATABASE` / `DROP DATABASE` | Crear / eliminar una base de datos completa |
| `CREATE TABLE` | Crear una tabla nueva (columnas, tipos, restricciones) |
| `ALTER TABLE ... ADD COLUMN` | Añadir una columna a una tabla existente |
| `ALTER TABLE ... MODIFY COLUMN` | Cambiar el tipo o las restricciones de una columna |
| `ALTER TABLE ... CHANGE COLUMN` | Cambiar el nombre (y la definición) de una columna |
| `ALTER TABLE ... DROP COLUMN` | Eliminar una columna |
| `ALTER TABLE ... ADD CONSTRAINT` | Añadir una restricción (`CHECK`, `UNIQUE`, `FOREIGN KEY`...) |
| `ALTER TABLE ... DROP CONSTRAINT` / `DROP FOREIGN KEY` | Eliminar una restricción |
| `ALTER TABLE ... RENAME TO` | Renombrar una tabla |
| `DROP TABLE` | Eliminar una tabla completa (estructura y datos) |
| `TRUNCATE TABLE` | Vaciar una tabla de todas sus filas |
| `CREATE INDEX` / `DROP INDEX` | Crear / eliminar un índice |
| `CREATE VIEW` / `DROP VIEW` | Crear / eliminar una vista |

---

## Base de datos de referencia

Este tema reutiliza el esquema y los datos de la base de datos **ACADEMIA**, descrita en detalle en "El lenguaje SQL: Manipulación" (sección "Base de datos de referencia: ACADEMIA"). El script `crear_bd_academia.sql` crea la base de datos con `CREATE TABLE`; el script `ddl_academia.sql` de este tema contiene, en el mismo orden que este documento, todas las instrucciones DDL de ejemplo (`ALTER TABLE`, `DROP TABLE`, `TRUNCATE TABLE`, `CREATE INDEX`, `CREATE VIEW`...), listas para ejecutarse a continuación.
