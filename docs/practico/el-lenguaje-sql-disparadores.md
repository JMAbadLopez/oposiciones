# El lenguaje SQL: Disparadores (Triggers)

## Objetivos

| # | Objetivo |
|---|---|
| 1 | Entender qué es un disparador (*trigger*) y cuándo tiene sentido usarlo. |
| 2 | Distinguir los disparadores por **evento** (`INSERT`/`UPDATE`/`DELETE`), por **momento** (`BEFORE`/`AFTER`) y por **ámbito** (`FOR EACH ROW`). |
| 3 | Usar `OLD` y `NEW` para acceder a los valores de la fila antes y después de la operación. |
| 4 | Construir ejemplos típicos: auditoría de altas, validación de datos, auditoría de cambios. |
| 5 | Reutilizar la base de datos ACADEMIA como esquema de referencia. |

!!! note "Sobre los datos de este documento"
    Los ejemplos parten de la base de datos ACADEMIA (`crear_bd_academia.sql`). Las instrucciones de este documento están recogidas en el script `plsql_academia.sql`, junto con los procedimientos y funciones del tema anterior.

!!! note "De dónde viene este tema"
    Este documento adapta a MySQL/MariaDB el contenido del manual de disparadores (Oracle) que ya tenías preparado. La estructura y la lógica de los ejemplos (auditoría, validación) son las mismas; cambia la sintaxis.

---

## Qué es un disparador

!!! tip "Idea clave"
    Un **disparador** (*trigger*) es un bloque de código asociado a una tabla que se ejecuta **automáticamente** cuando ocurre un evento determinado sobre esa tabla (`INSERT`, `UPDATE` o `DELETE`). A diferencia de un procedimiento, **nunca se llama explícitamente**: salta solo, como reacción al evento.

Casos de uso típicos:

- **Auditoría**: registrar en una tabla aparte quién hizo qué y cuándo (altas, bajas, cambios).
- **Validación adicional**: rechazar una operación si no cumple una regla de negocio que un `CHECK` no puede expresar fácilmente.
- **Mantenimiento de datos derivados**: actualizar un contador o un total cada vez que cambian los datos de los que depende.

---

## Tipos de disparador

Un disparador se clasifica por tres ejes independientes:

| Eje | Valores posibles | Significado |
|---|---|---|
| Evento | `INSERT`, `UPDATE`, `DELETE` | Qué operación lo activa |
| Momento | `BEFORE`, `AFTER` | Si se ejecuta antes o después de que la operación se aplique |
| Ámbito | `FOR EACH ROW` | Se ejecuta una vez por cada fila afectada (es el único ámbito que admite MySQL/MariaDB) |

!!! note "MySQL solo admite disparadores por fila"
    Oracle permite también disparadores `FOR EACH STATEMENT` (una sola ejecución por toda la instrucción, sin importar cuántas filas afecte). **MySQL y MariaDB no tienen esa opción**: todos los disparadores son `FOR EACH ROW`, aunque la cláusula hay que escribirla igualmente por sintaxis.

### Sintaxis básica

```sql
CREATE TRIGGER nombre_disparador
{BEFORE | AFTER} {INSERT | UPDATE | DELETE} ON nombre_tabla
FOR EACH ROW
BEGIN
    -- cuerpo del disparador
END;
```

Al igual que los procedimientos, si el cuerpo tiene más de una instrucción hace falta `DELIMITER //` antes de `CREATE TRIGGER` y `DELIMITER ;` después.

### `OLD` y `NEW`

Dentro del cuerpo de un disparador se puede acceder a los valores de la fila con dos pseudo-registros:

| | Disponible en | Significado |
|---|---|---|
| `OLD.columna` | `UPDATE`, `DELETE` | Valor de la columna **antes** de la operación |
| `NEW.columna` | `INSERT`, `UPDATE` | Valor de la columna **después** de la operación |

Es decir: en un `INSERT` solo existe `NEW` (la fila no existía antes); en un `DELETE` solo existe `OLD` (la fila deja de existir); en un `UPDATE` existen ambos.

!!! note "Diferencia de sintaxis con Oracle"
    Oracle usa `:OLD.columna` y `:NEW.columna` (con dos puntos, porque son variables ligadas). MySQL/MariaDB usa `OLD.columna` y `NEW.columna`, sin los dos puntos.

---

## Ejemplo 1: auditoría de altas (`AFTER INSERT`)

Se quiere registrar, en una tabla aparte, cada nota que se da de alta.

```sql
CREATE TABLE Auditoria_Nota (
    idaudit    INT NOT NULL AUTO_INCREMENT,
    nia        INT,
    codmodulo  INT,
    nota       DECIMAL(4,2),
    fecha_alta DATETIME,
    PRIMARY KEY (idaudit)
);

DELIMITER //
CREATE TRIGGER trg_auditoria_nota_insert
AFTER INSERT ON Nota
FOR EACH ROW
BEGIN
    INSERT INTO Auditoria_Nota (nia, codmodulo, nota, fecha_alta)
    VALUES (NEW.nia, NEW.codmodulo, NEW.nota, NOW());
END //
DELIMITER ;

INSERT INTO Nota (nia, codmodulo, nota) VALUES (3, 2, 7.5);
SELECT * FROM Auditoria_Nota;
```

```text
+---------+-----+-----------+------+---------------------+
| idaudit | nia | codmodulo | nota | fecha_alta          |
+---------+-----+-----------+------+---------------------+
|       1 |   3 |         2 | 7.50 | 2026-10-16 14:07:26 |
+---------+-----+-----------+------+---------------------+
```

Se usa `AFTER INSERT` (no `BEFORE`) porque el registro de auditoría solo tiene sentido **una vez que la inserción ya se ha confirmado**. Este mismo patrón es el que aparece en el ejercicio de la biblioteca (Examen 2025), donde un disparador `AFTER DELETE`/`AFTER UPDATE` sobre `LIBROS` registra los cambios en una tabla de auditoría.

---

## Ejemplo 2: validación de datos (`BEFORE INSERT`)

Se quiere rechazar una inserción si la nota está fuera del rango 0–10. Para forzarlo "a mano" (como haría un disparador de validación, aunque ACADEMIA ya tenga esta regla como `CHECK` declarativo):

```sql
DELIMITER //
CREATE TRIGGER trg_nota_before_insert_valida
BEFORE INSERT ON Nota
FOR EACH ROW
BEGIN
    IF NEW.nota < 0 OR NEW.nota > 10 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'La nota debe estar entre 0 y 10';
    END IF;
END //
DELIMITER ;

INSERT INTO Nota (nia, codmodulo, nota) VALUES (4, 2, 23);
```

```text
ERROR 1644 (45000): La nota debe estar entre 0 y 10
```

Se usa `BEFORE INSERT` porque la validación debe ocurrir **antes** de que la fila se escriba: si el disparador lanza un `SIGNAL`, toda la operación se cancela y la fila no llega a insertarse.

!!! note "Relación con `CHECK`"
    Desde MySQL 8.0.16 y MariaDB 10.2.1, las restricciones `CHECK` declarativas (como `ck_nota_rango` en la tabla `Nota` de ACADEMIA) cubren la mayoría de validaciones simples de este tipo, y son preferibles por ser más directas. Un disparador de validación sigue siendo útil cuando la regla es más compleja que una simple comparación (por ejemplo, si depende de datos de **otra** tabla) o en versiones de MySQL anteriores a esas.

---

## Ejemplo 3: auditoría de cambios con `OLD` y `NEW` (`BEFORE UPDATE`)

Se quiere guardar, cada vez que se modifica una nota, tanto el valor anterior como el nuevo.

```sql
CREATE TABLE Cambios_Nota (
    idcambio      INT NOT NULL AUTO_INCREMENT,
    nia           INT,
    codmodulo     INT,
    nota_anterior DECIMAL(4,2),
    nota_nueva    DECIMAL(4,2),
    fecha_cambio  DATETIME,
    PRIMARY KEY (idcambio)
);

DELIMITER //
CREATE TRIGGER trg_nota_before_update
BEFORE UPDATE ON Nota
FOR EACH ROW
BEGIN
    INSERT INTO Cambios_Nota (nia, codmodulo, nota_anterior, nota_nueva, fecha_cambio)
    VALUES (OLD.nia, OLD.codmodulo, OLD.nota, NEW.nota, NOW());
END //
DELIMITER ;

UPDATE Nota SET nota = 8.0 WHERE nia = 3 AND codmodulo = 2;
SELECT * FROM Cambios_Nota;
```

```text
+-----------+-----+-----------+---------------+------------+---------------------+
| idcambio  | nia | codmodulo | nota_anterior | nota_nueva | fecha_cambio        |
+-----------+-----+-----------+---------------+------------+---------------------+
|         1 |   3 |         2 |          7.50 |       8.00 | 2026-10-16 14:07:26 |
+-----------+-----+-----------+---------------+------------+---------------------+
```

Aquí se usa `BEFORE UPDATE` porque, en el momento en que se ejecuta el disparador, todavía se dispone de `OLD` (el valor que tenía la fila) y ya se conoce `NEW` (el valor que va a tomar); da igual que sea `BEFORE` o `AFTER` para este caso concreto, porque ninguno de los dos cancela nada — la elección habitual es `BEFORE` cuando solo se audita, y se reserva `AFTER` para acciones que deban ver el estado ya confirmado en la tabla.

---

## Gestionar disparadores: deshabilitar y eliminar

MySQL/MariaDB **no tiene** una instrucción para deshabilitar temporalmente un disparador sin eliminarlo (a diferencia de Oracle, que sí ofrece `ALTER TRIGGER ... DISABLE`). La única forma de "desactivarlo" es eliminarlo y, si se necesita de nuevo, volver a crearlo:

```sql
SHOW TRIGGERS FROM academia;

DROP TRIGGER IF EXISTS trg_nota_before_insert_valida;
```

!!! note "Diferencias con PL/SQL (Oracle)"
    - Oracle: `ALTER TRIGGER nombre DISABLE;` / `ALTER TRIGGER nombre ENABLE;`. MySQL/MariaDB: no existe equivalente — solo `DROP TRIGGER` y volver a crearlo.
    - Oracle: `:OLD.columna` / `:NEW.columna`. MySQL/MariaDB: `OLD.columna` / `NEW.columna` (sin los dos puntos).
    - Oracle admite disparadores `FOR EACH STATEMENT` además de `FOR EACH ROW`; MySQL/MariaDB solo admite `FOR EACH ROW`.

---

## Resumen

| Momento | Cuándo usarlo | `OLD` disponible | `NEW` disponible |
|---|---|---|---|
| `BEFORE INSERT` | Validar o modificar datos antes de guardarlos | No | Sí |
| `AFTER INSERT` | Auditoría, acciones una vez confirmada el alta | No | Sí |
| `BEFORE UPDATE` | Validar el cambio, o auditar antes de aplicarlo | Sí | Sí |
| `AFTER UPDATE` | Acciones que necesiten ver el dato ya actualizado | Sí | Sí |
| `BEFORE DELETE` | Validar si se puede borrar, o archivar antes de perder el dato | Sí | No |
| `AFTER DELETE` | Auditoría de bajas, limpieza de datos relacionados | Sí | No |
