# El lenguaje SQL: Manipulación II

## Objetivos

| # | Objetivo |
|---|---|
| 1 | Completar el Lenguaje de Manipulación de Datos (DML) visto en el tema anterior, centrado en `SELECT`, con las instrucciones que **modifican** datos: `INSERT`, `UPDATE` y `DELETE`. |
| 2 | Ver ejemplos prácticos de inserción, actualización y borrado de filas, incluyendo los errores más habituales por violación de restricciones. |
| 3 | Reutilizar la base de datos ACADEMIA como esquema de referencia, para dar continuidad a los temas anteriores. |

!!! note "Sobre los datos de este documento"
    Los ejemplos de este tema parten de la misma base de datos ACADEMIA usada en los temas anteriores (ver `crear_bd_academia.sql`). Las instrucciones de este documento están recogidas en el script `dml2_academia.sql`, pensado para ejecutarse **después** de `crear_bd_academia.sql`.

    A diferencia del tema de consultas (`SELECT`), aquí cada ejemplo **modifica** los datos, así que el resultado de un ejemplo puede depender de los anteriores (se indica en cada caso). Si después se quiere volver a usar `consultas_teoria.sql` o `ddl_academia.sql` con los datos originales, hay que volver a ejecutar `crear_bd_academia.sql`.

---

## INSERT, UPDATE y DELETE dentro del DML

!!! tip "Idea clave"
    El Lenguaje de Manipulación de Datos (DML) no se limita a consultar: también permite modificar el contenido de las tablas sin tocar su estructura (eso es tarea del DDL, visto en el tema anterior).

| Instrucción | Qué hace |
|---|---|
| `INSERT INTO` | Añade una o varias filas nuevas a una tabla |
| `UPDATE` | Modifica el valor de una o varias columnas en las filas que cumplen una condición |
| `DELETE FROM` | Elimina las filas que cumplen una condición |

Las tres respetan las restricciones definidas con DDL (`PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`, `CHECK`, `NOT NULL`): si una instrucción las viola, se rechaza con un error y no se aplica ningún cambio.

---

## INSERT INTO

### Insertar una fila indicando todas las columnas

!!! tip "Idea clave"
    La forma más explícita de `INSERT` indica el nombre de cada columna y su valor correspondiente, en el mismo orden. No es obligatorio seguir el orden de la tabla, pero si se nombran las columnas, cada valor debe corresponder a la columna indicada en esa misma posición.

**EJEMPLO:** Añadir un nuevo módulo.

```sql
INSERT INTO Modulo (codmodulo, nombre, horas, profesor)
VALUES (7, 'Programación de Servicios y Procesos', 133, 'Marta Ibáñez');
```

**Resultado:** una fila nueva en `Modulo`:

| codmodulo | nombre | horas | profesor |
|---|---|---|---|
| 7 | Programación de Servicios y Procesos | 133 | Marta Ibáñez |

### Insertar indicando solo algunas columnas

!!! tip "Idea clave"
    No hace falta dar un valor a todas las columnas: las que se omiten toman su valor por defecto (`DEFAULT`, si se definió alguno) o `NULL` (si la columna lo admite). Si una columna omitida es `NOT NULL` y no tiene `DEFAULT`, el `INSERT` falla.

**EJEMPLO:** Añadir un alumno sin indicar su edad.

```sql
INSERT INTO Alumno (nia, nombre, codciclo)
VALUES (11, 'Elena Bort', 'DAM');
```

**Resultado:** `edad` no se ha indicado y la columna admite `NULL` (no tiene `NOT NULL`), así que toma ese valor:

| nia | nombre | edad | codciclo |
|---|---|---|---|
| 11 | Elena Bort | `NULL` | DAM |

### Insertar varias filas en una sola instrucción

!!! tip "Idea clave"
    `INSERT` admite varias listas de valores separadas por comas, en una sola instrucción. Es más eficiente que repetir un `INSERT` por cada fila, porque el motor solo tiene que validar y aplicar el cambio una vez.

**EJEMPLO:** Añadir dos alumnos a la vez.

```sql
INSERT INTO Alumno (nia, nombre, edad, codciclo) VALUES
    (12, 'Marcos Ibáñez', 24, 'DAW'),
    (13, 'Sara Peña',     21, 'ASIR');
```

**Resultado:** dos filas nuevas en `Alumno` (Marcos Ibáñez, DAW, 24 años; Sara Peña, ASIR, 21 años).

### INSERT ... SELECT

!!! tip "Idea clave"
    En vez de una lista de valores literales, `INSERT` puede tomar como origen el resultado de una consulta `SELECT`. Las columnas de la consulta deben coincidir en número y en tipo (compatible) con las columnas de destino, aunque no hace falta que se llamen igual.

**EJEMPLO:** Copiar a una tabla aparte los datos básicos de los alumnos de DAM.

```sql
CREATE TABLE Alumno_DAM (nia INT, nombre VARCHAR(60), edad INT);

INSERT INTO Alumno_DAM
    SELECT nia, nombre, edad FROM Alumno WHERE codciclo = 'DAM';
```

**Resultado:** `Alumno_DAM` recibe 5 filas: los 4 alumnos originales de DAM (Antonio Ruiz, Javier Torres, Carlos Ferrer, Manuel Ortega) más Elena Bort, incorporada en el ejemplo anterior.

### Errores por violación de restricciones

!!! tip "Idea clave"
    `INSERT` respeta todas las restricciones de la tabla. Los dos errores más habituales en examen son la clave primaria duplicada y la clave ajena que no existe en la tabla referenciada.

**EJEMPLO:** Intentar insertar un alumno con un `nia` ya existente.

```sql
INSERT INTO Alumno (nia, nombre, edad, codciclo) VALUES (1, 'Duplicado', 30, 'DAM');
```

**Resultado:** error, porque `nia = 1` ya existe (es la clave primaria de Antonio Ruiz):

```
ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY'
```

**EJEMPLO:** Intentar insertar un alumno con un ciclo que no existe.

```sql
INSERT INTO Alumno (nia, nombre, edad, codciclo) VALUES (14, 'Nuevo', 20, 'XXX');
```

**Resultado:** error, porque `'XXX'` no existe como `codciclo` en la tabla `Ciclo`, y `Alumno.codciclo` es una clave ajena hacia ella:

```
ERROR 1452 (23000): Cannot add or update a child row: a foreign key
constraint fails (`academia`.`Alumno`, CONSTRAINT `fk_alumno_ciclo`
FOREIGN KEY (`codciclo`) REFERENCES `Ciclo` (`codciclo`))
```

---

## UPDATE

### Modificación simple con WHERE

!!! tip "Idea clave"
    `UPDATE tabla SET columna = valor WHERE condición` cambia el valor de una o varias columnas en las filas que cumplen la condición. Igual que en `DELETE`, si se omite el `WHERE`, se actualizan **todas** las filas de la tabla.

**EJEMPLO:** Corregir la edad de Jorge Salas, que no se había registrado.

```sql
UPDATE Alumno SET edad = 25 WHERE nia = 10;
```

**Resultado:**

| nia | nombre | edad | codciclo |
|---|---|---|---|
| 10 | Jorge Salas | 25 | SMR |

### Modificación con una expresión sobre la propia columna

!!! tip "Idea clave"
    El valor nuevo puede calcularse a partir del valor actual de la fila, igual que en `SELECT`. Es la forma habitual de incrementar, reducir o recalcular un valor sin tener que consultarlo antes por separado.

**EJEMPLO:** Revalorizar un 5 % el importe de la Beca de Excelencia.

```sql
UPDATE Distincion SET importe = importe * 1.05 WHERE tipo = 'Beca de Excelencia';
```

**Resultado:** el importe pasa de 60,00 € a **63,00 €**.

### Condición con subconsulta

!!! tip "Idea clave"
    La condición del `WHERE` de un `UPDATE` admite subconsultas, igual que en `SELECT` o `DELETE`. Es útil cuando la condición depende de datos de otra tabla.

**EJEMPLO:** Sumar un año a la edad de todos los alumnos del ciclo cuyo tutor es Álvaro Pino.

```sql
UPDATE Alumno SET edad = edad + 1
WHERE codciclo = (SELECT codciclo FROM Ciclo WHERE tutor = 'Álvaro Pino');
```

**Resultado:** Álvaro Pino es tutor de SMR, así que afecta a sus dos alumnos:

| nia | nombre | edad (antes) | edad (después) |
|---|---|---|---|
| 9 | Diego Vidal | 17 | 18 |
| 10 | Jorge Salas | 25 *(ya corregida en el ejemplo anterior)* | 26 |

!!! warning "UPDATE sin WHERE"
    `UPDATE Modulo SET horas = horas + 10;` (sin `WHERE`) sumaría 10 horas a **todos** los módulos, no a uno en concreto. Es el mismo tipo de error, y el mismo riesgo, que un `DELETE` sin `WHERE`: conviene comprobar siempre la condición con un `SELECT` antes de lanzar el `UPDATE`.

---

## DELETE

### Borrado con WHERE

!!! tip "Idea clave"
    `DELETE FROM tabla WHERE condición` elimina las filas que cumplen la condición. A diferencia de `TRUNCATE TABLE` (DDL, visto en el tema anterior), sí admite `WHERE` —puede borrar solo algunas filas— y sí se puede deshacer con `ROLLBACK` dentro de una transacción.

**EJEMPLO:** Eliminar la nota de Luis Campos en el módulo de Bases de Datos.

```sql
DELETE FROM Nota WHERE nia = 5 AND codmodulo = 3;
```

**Resultado:** la fila `(5, 3, 5.00)` desaparece de `Nota`; a Luis Campos le quedan notas en los módulos 2, 4 y 6.

**EJEMPLO:** Eliminar a un alumno sin filas dependientes en otras tablas.

```sql
DELETE FROM Alumno WHERE nia = 13;
```

**Resultado:** Sara Peña (añadida en un ejemplo anterior, sin notas ni distinciones) se elimina sin problema.

### Bloqueado por integridad referencial

!!! tip "Idea clave"
    Si una fila está referenciada por una clave ajena de otra tabla (y esa clave ajena no tiene `ON DELETE CASCADE` ni `ON DELETE SET NULL`), `DELETE` la rechaza: primero hay que borrar (o reasignar) las filas dependientes.

**EJEMPLO:** Intentar eliminar a Antonio Ruiz, que tiene notas registradas.

```sql
DELETE FROM Alumno WHERE nia = 1;
```

**Resultado:** error, porque `Nota.nia` es una clave ajena hacia `Alumno.nia` (sin `CASCADE`) y Antonio Ruiz tiene filas en `Nota`:

```
ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key
constraint fails (`academia`.`Nota`, CONSTRAINT `fk_nota_alumno`
FOREIGN KEY (`nia`) REFERENCES `Alumno` (`nia`))
```

### Condición con subconsulta

!!! tip "Idea clave"
    Igual que en `UPDATE`, la condición de un `DELETE` puede apoyarse en una subconsulta sobre otra tabla.

**EJEMPLO:** Eliminar todas las distinciones obtenidas cuyo importe sea inferior a 70 €.

```sql
DELETE FROM Obtener_Distincion
WHERE codigo IN (SELECT codigo FROM Distincion WHERE importe < 70);
```

**Resultado:** la Beca de Excelencia (D4) tiene un importe de 63,00 € (tras la revalorización del ejemplo anterior), por debajo de 70 €. Se eliminan sus 5 filas en `Obtener_Distincion` (las de Diego Vidal y Jorge Salas).

### DELETE sin WHERE frente a TRUNCATE TABLE

!!! tip "Idea clave"
    `DELETE FROM tabla` (sin `WHERE`) y `TRUNCATE TABLE tabla` dejan el mismo resultado visible —la tabla vacía—, pero no son la misma instrucción: una es DML y la otra DDL, con diferencias importantes (ver la tabla comparativa del tema anterior).

**EJEMPLO:** sobre una tabla copia, para no perder los datos reales de `Nota`.

```sql
CREATE TABLE Nota_Copia LIKE Nota;
INSERT INTO Nota_Copia SELECT * FROM Nota;

DELETE FROM Nota_Copia;
```

**Resultado:** `Nota_Copia` tenía 19 filas (20 originales menos la eliminada en el primer ejemplo de `DELETE`); tras el `DELETE` sin `WHERE`, 0 filas. El efecto visible es idéntico al de un `TRUNCATE TABLE Nota_Copia`, pero aquí sí podría deshacerse con `ROLLBACK` si se hubiera ejecutado dentro de una transacción, y no reinicia ningún contador `AUTO_INCREMENT` (esta tabla no tiene ninguno).

---

## Resumen: INSERT / UPDATE / DELETE

| | `INSERT INTO` | `UPDATE` | `DELETE FROM` |
|---|---|---|---|
| Categoría | DML | DML | DML |
| Efecto | Añade filas nuevas | Modifica columnas de filas existentes | Elimina filas existentes |
| Admite `WHERE` | No aplica (no hay filas que filtrar) | Sí (opcional, pero casi siempre necesario) | Sí (opcional, pero casi siempre necesario) |
| Riesgo típico | Violar `PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY` o `CHECK` | Olvidar el `WHERE` y modificar todas las filas | Olvidar el `WHERE` y borrar todas las filas, o chocar con una clave ajena `RESTRICT` |
| Se puede deshacer con `ROLLBACK` | Sí (dentro de una transacción) | Sí (dentro de una transacción) | Sí (dentro de una transacción) |

---

## Base de datos de referencia

Este tema reutiliza el esquema y los datos de la base de datos **ACADEMIA**, descrita en detalle en "El lenguaje SQL: Manipulación" (sección "Base de datos de referencia: ACADEMIA"). El script `crear_bd_academia.sql` crea y puebla la base de datos; el script `dml2_academia.sql` de este tema contiene, en el mismo orden que este documento, todas las instrucciones `INSERT`, `UPDATE` y `DELETE` de ejemplo, listas para ejecutarse a continuación.
