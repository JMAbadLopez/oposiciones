# El lenguaje SQL: Procedimientos, funciones y cursores

## Objetivos

| # | Objetivo |
|---|---|
| 1 | Entender qué es una **rutina almacenada** (procedimiento o función) y por qué ampliar SQL con estructuras de control. |
| 2 | Conocer la estructura básica de un bloque de código en MySQL/MariaDB: variables, `IF`/`CASE`, bucles (`WHILE`, `LOOP`, `FOR`). |
| 3 | Crear y usar **procedimientos almacenados** (`CREATE PROCEDURE`) con parámetros `IN`, `OUT` e `INOUT`. |
| 4 | Crear y usar **funciones almacenadas** (`CREATE FUNCTION`), y distinguirlas de los procedimientos. |
| 5 | Recorrer un conjunto de resultados fila a fila con **cursores explícitos**. |
| 6 | Reutilizar la base de datos ACADEMIA como esquema de referencia. |

!!! note "Sobre los datos de este documento"
    Los ejemplos parten de la base de datos ACADEMIA (`crear_bd_academia.sql`), ya usada en los temas anteriores. Todas las rutinas de este documento están recogidas en el script `plsql_academia.sql`, pensado para ejecutarse **después** de `crear_bd_academia.sql`.

---

## Por qué SQL "puro" no basta

!!! tip "Idea clave"
    SQL (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) es un lenguaje **declarativo**: describe *qué* datos se quieren, no *cómo* obtenerlos paso a paso. Hay tareas que necesitan lógica **procedimental** — variables, condicionales, bucles, control de errores — y para eso MySQL/MariaDB ofrece un lenguaje de extensión que permite agrupar esa lógica en **rutinas almacenadas**: procedimientos y funciones.

| | SQL declarativo | Rutina almacenada |
|---|---|---|
| Qué expresa | Una consulta o modificación de datos | Una secuencia de pasos con lógica de control |
| Ejemplo | `SELECT * FROM Alumno WHERE edad > 18` | "Por cada alumno sin nota, insertar un aviso, y si falla algo, deshacerlo todo" |
| Se guarda en la BD | No (se ejecuta y se olvida) | Sí, con un nombre, para poder invocarla después |

Las rutinas almacenadas se escriben una vez, se guardan en el servidor con `CREATE PROCEDURE` / `CREATE FUNCTION`, y se invocan tantas veces como se necesite (`CALL`, o directamente en una consulta si es una función). Esto evita repetir la misma lógica en cada aplicación cliente y la centraliza en la base de datos.

---

## El delimitador `DELIMITER`

!!! warning "Un problema puramente del cliente de MySQL"
    El cuerpo de una rutina casi siempre contiene varias instrucciones SQL, cada una terminada en `;`. Pero el cliente de línea de comandos de MySQL usa `;` para saber dónde acaba **cada orden que le envías**. Si no se le avisa, al llegar al primer `;` interno pensaría que el `CREATE PROCEDURE` ya ha terminado, y daría un error de sintaxis.

La solución es cambiar temporalmente el delimitador con `DELIMITER //` (o cualquier otro símbolo que no aparezca en el código), escribir toda la rutina terminada en `//`, y devolver el delimitador a `;` al final:

```sql
DELIMITER //
CREATE PROCEDURE ejemplo()
BEGIN
    SELECT 'hola';
END //
DELIMITER ;
```

Esto **no es sintaxis SQL**: es una instrucción del cliente (`mysql`, phpMyAdmin suele gestionarlo solo). Si se ejecuta el script desde otra herramienta puede no ser necesario, pero conviene incluirlo siempre por compatibilidad.

---

## Bloques, variables y estructuras de control

!!! tip "Idea clave"
    Dentro de un `BEGIN ... END` se pueden declarar variables locales con `DECLARE` y usar estructuras de control (`IF`, `CASE`, bucles). Es el equivalente, en MySQL/MariaDB, al bloque `DECLARE ... BEGIN ... END;` de PL/SQL — con una diferencia importante: **en MySQL este bloque solo existe dentro de una rutina almacenada** (procedimiento, función o disparador). No hay un "modo anónimo" para ejecutar un bloque suelto desde el cliente como sí permite Oracle (`DECLARE ... BEGIN ... END; /`).

### Variables

```sql
DECLARE v_total INT DEFAULT 0;
DECLARE v_nombre VARCHAR(60);
DECLARE v_nota DECIMAL(4,2) DEFAULT 0.0;
```

`DECLARE` siempre va al principio del bloque, antes de cualquier otra instrucción. `SET` asigna un valor; `SELECT ... INTO ...` asigna el resultado de una consulta:

```sql
SET v_total = v_total + 1;
SELECT AVG(nota) INTO v_nota FROM Nota WHERE nia = 1;
```

### Condicionales

```sql
IF v_nota >= 5 THEN
    SELECT 'Aprobado' AS resultado;
ELSEIF v_nota >= 4 THEN
    SELECT 'Cerca del aprobado' AS resultado;
ELSE
    SELECT 'Suspenso' AS resultado;
END IF;
```

```sql
CASE
    WHEN v_nota >= 9 THEN SELECT 'Sobresaliente';
    WHEN v_nota >= 7 THEN SELECT 'Notable';
    WHEN v_nota >= 5 THEN SELECT 'Suficiente';
    ELSE SELECT 'Suspenso';
END CASE;
```

### Bucles

MySQL/MariaDB ofrece tres formas de repetir código: `WHILE`, `LOOP` y (solo en MariaDB) `FOR`.

```sql
-- WHILE: se repite mientras la condición sea verdadera
DECLARE i INT DEFAULT 1;
DECLARE v_total INT DEFAULT 0;
WHILE i <= 5 DO
    SET v_total = v_total + i;
    SET i = i + 1;
END WHILE;
```

```sql
-- LOOP: bucle "infinito" que se interrumpe con LEAVE (necesario para recorrer cursores, ver más abajo)
DECLARE i INT DEFAULT 1;
bucle: LOOP
    IF i > 5 THEN
        LEAVE bucle;
    END IF;
    SET i = i + 1;
END LOOP;
```

!!! note "El bucle `FOR` es una extensión de MariaDB"
    MariaDB (desde la versión 10.3) admite `FOR i IN 1..5 DO ... END FOR;`, muy parecido al `FOR` de Oracle. **MySQL no tiene esta sintaxis**: si el ejercicio puede ejecutarse en MySQL "a pelo", es más seguro usar `WHILE`, que funciona igual en ambos motores. Los dos bloques siguientes son equivalentes:

    ```sql
    -- Solo MariaDB
    FOR i IN 1..5 DO
        SET v_total = v_total + i;
    END FOR;
    ```

    ```sql
    -- MySQL y MariaDB
    SET i = 1;
    WHILE i <= 5 DO
        SET v_total = v_total + i;
        SET i = i + 1;
    END WHILE;
    ```

---

## Procedimientos almacenados

!!! tip "Idea clave"
    Un **procedimiento** (`CREATE PROCEDURE`) agrupa una secuencia de instrucciones bajo un nombre, para poder invocarla con `CALL`. No devuelve un valor con `RETURN` (a diferencia de una función): si necesita "devolver" algo, lo hace a través de parámetros `OUT`/`INOUT`, o simplemente ejecutando un `SELECT` que el cliente recibe como resultado.

### Tipos de parámetro

| Modo | Significado |
|---|---|
| `IN` (por defecto) | El procedimiento recibe un valor, pero no puede modificar la variable del llamador. |
| `OUT` | El procedimiento no recibe nada útil de esa variable, pero le asigna un valor que el llamador podrá leer después de la llamada. |
| `INOUT` | Combina ambos: el procedimiento recibe un valor inicial y puede modificarlo; el llamador ve el valor actualizado. |

### Ejemplo simple: procedimiento con solo parámetros `IN`

```sql
DELIMITER //
CREATE PROCEDURE insertar_nota(IN p_nia INT, IN p_codmodulo INT, IN p_nota DECIMAL(4,2))
BEGIN
    INSERT INTO Nota (nia, codmodulo, nota) VALUES (p_nia, p_codmodulo, p_nota);
END //
DELIMITER ;

CALL insertar_nota(6, 3, 7.5);
```

### Ejemplo con parámetro `OUT`

Las variables que recogen un `OUT` se declaran en la sesión del cliente con `@` (variables de usuario), no con `DECLARE` (que solo vale dentro de una rutina):

```sql
DELIMITER //
CREATE PROCEDURE contar_notas_alumno(IN p_nia INT, OUT p_total INT)
BEGIN
    SELECT COUNT(*) INTO p_total FROM Nota WHERE nia = p_nia;
END //
DELIMITER ;

CALL contar_notas_alumno(1, @total);
SELECT @total;
```

```text
+--------+
| @total |
+--------+
|      4 |
+--------+
```

### Ejemplo con `OUT` y `ROW_COUNT()`

`ROW_COUNT()` devuelve el número de filas afectadas por la última instrucción `INSERT`/`UPDATE`/`DELETE` — es el equivalente MySQL/MariaDB de `SQL%ROWCOUNT` en PL/SQL:

```sql
DELIMITER //
CREATE PROCEDURE actualizar_nota(
    IN  p_nia INT,
    IN  p_codmodulo INT,
    IN  p_nueva_nota DECIMAL(4,2),
    OUT p_filas_afectadas INT
)
BEGIN
    UPDATE Nota SET nota = p_nueva_nota WHERE nia = p_nia AND codmodulo = p_codmodulo;
    SET p_filas_afectadas = ROW_COUNT();
END //
DELIMITER ;

CALL actualizar_nota(1, 2, 6.2, @filas);
SELECT @filas;   -- 1 si el alumno 1 tenía nota en el módulo 2, 0 si no existía esa fila
```

### Procedimientos anidados (uno llama a otro)

Un procedimiento puede invocar a otro con `CALL`, igual que una consulta normal:

```sql
DELIMITER //
CREATE PROCEDURE informe_alumno(IN p_nia INT)
BEGIN
    DECLARE v_total INT;
    CALL contar_notas_alumno(p_nia, v_total);
    SELECT p_nia AS nia, v_total AS total_notas;
END //
DELIMITER ;

CALL informe_alumno(5);
```

!!! note "Diferencias con PL/SQL (Oracle)"
    - La sintaxis de cabecera es distinta: Oracle usa `CREATE OR REPLACE PROCEDURE nombre (param IN/OUT/IN OUT tipo) AS BEGIN ... END;`; MySQL usa `CREATE PROCEDURE nombre (IN/OUT/INOUT param tipo) BEGIN ... END`.
    - Oracle llama a un procedimiento con `EXEC nombre(...)` (en SQL*Plus) o simplemente `nombre(...)` dentro de otro bloque; MySQL siempre usa `CALL nombre(...)`.
    - El equivalente de `SQL%ROWCOUNT` en MySQL es la función `ROW_COUNT()`.

---

## Funciones almacenadas

!!! tip "Idea clave"
    Una **función** (`CREATE FUNCTION`) siempre devuelve un único valor con `RETURN`, y puede usarse dentro de una consulta SQL como si fuera una función predefinida (`SELECT nombre_funcion(...)`). A cambio de esa ventaja, MySQL impone una restricción que Oracle no tiene: **una función solo puede tener parámetros `IN`** (no admite `OUT` ni `INOUT`).

| | Procedimiento | Función |
|---|---|---|
| Se invoca con | `CALL nombre(...)` | Dentro de una expresión: `SELECT nombre(...)` |
| Devuelve valor | No directamente (usa `OUT`) | Sí, con `RETURN`, tipado por `RETURNS` |
| Parámetros | `IN`, `OUT`, `INOUT` | Solo `IN` |
| Uso típico | Tareas con varios pasos, efectos (INSERT/UPDATE) | Cálculos reutilizables dentro de consultas |

### Ejemplo: función con un parámetro

```sql
DELIMITER //
CREATE FUNCTION nota_media_alumno(p_nia INT)
RETURNS DECIMAL(4,2)
BEGIN
    DECLARE v_media DECIMAL(4,2);
    SELECT AVG(nota) INTO v_media FROM Nota WHERE nia = p_nia;
    RETURN v_media;
END //
DELIMITER ;

SELECT nia, nombre, nota_media_alumno(nia) AS media
FROM Alumno
WHERE nia IN (1, 2, 5);
```

```text
+-----+---------------+-------+
| nia | nombre        | media |
+-----+---------------+-------+
|   1 | Antonio Ruiz  |  6.38 |
|   2 | Javier Torres |  6.63 |
|   5 | Luis Campos   |  5.50 |
+-----+---------------+-------+
```

!!! note "Un detalle que te puedes encontrar"
    Si en algún servidor la creación de la función da este error:

    ```text
    ERROR 1418 (HY000): This function has none of DETERMINISTIC, NO SQL, or
    READS SQL DATA in its declaration and binary logging is enabled
    (you *might* want to use the less safe log_bin_trust_function_creators variable)
    ```

    es porque ese servidor tiene activado el registro binario (*binary log*) y, por seguridad, exige que toda función declare si es determinista y si lee o modifica datos. Dos formas de resolverlo:

    **Opción 1 (recomendada): añadir la cláusula que falta a la función**

    ```sql
    CREATE FUNCTION nota_media_alumno(p_nia INT)
    RETURNS DECIMAL(4,2)
    DETERMINISTIC
    READS SQL DATA
    BEGIN
        ...
    END
    ```

    **Opción 2: relajar la restricción en el servidor** (solo si no se puede tocar la función, por ejemplo en un servidor ya en producción)

    ```sql
    SET GLOBAL log_bin_trust_function_creators = 1;
    ```

    La opción 1 es preferible porque documenta el comportamiento real de la función; la opción 2 afecta a todo el servidor y a cualquier función que se cree en él, no solo a la nuestra.

### Función sin parámetros de entrada

```sql
DELIMITER //
CREATE FUNCTION total_alumnos()
RETURNS INT
BEGIN
    DECLARE v_total INT;
    SELECT COUNT(*) INTO v_total FROM Alumno;
    RETURN v_total;
END //
DELIMITER ;

SELECT total_alumnos();
```

### Funciones anidadas

Una función puede llamar a otra, igual que en SQL normal:

```sql
DELIMITER //
CREATE FUNCTION nota_aprobada(p_nota DECIMAL(4,2))
RETURNS BOOLEAN
BEGIN
    RETURN p_nota >= 5;
END //

CREATE FUNCTION contar_aprobados_modulo(p_codmodulo INT)
RETURNS INT
BEGIN
    DECLARE v_total INT;
    SELECT COUNT(*) INTO v_total
    FROM Nota
    WHERE codmodulo = p_codmodulo AND nota_aprobada(nota);
    RETURN v_total;
END //
DELIMITER ;

SELECT contar_aprobados_modulo(2);
```

!!! note "Diferencias con PL/SQL (Oracle)"
    - Oracle permite funciones con parámetros `OUT`/`IN OUT`; MySQL **no** — si se necesita devolver varios valores, hay que usar un procedimiento con parámetros `OUT`.
    - Oracle: `CREATE OR REPLACE FUNCTION nombre (...) RETURN tipo AS BEGIN ... RETURN valor; END;`. MySQL: `CREATE FUNCTION nombre (...) RETURNS tipo [DETERMINISTIC] BEGIN ... RETURN valor; END`.
    - `DBMS_OUTPUT.PUT_LINE(...)`, muy usado en Oracle para depurar, no tiene equivalente directo dentro de una rutina MySQL: para "ver" un valor intermedio se suele usar un `SELECT` de depuración o insertarlo en una tabla auxiliar.

---

## Cursores explícitos

!!! tip "Idea clave"
    Un **cursor** permite recorrer el resultado de una consulta **fila a fila**, cuando no basta con que SQL procese el conjunto de golpe (por ejemplo, para aplicar una lógica distinta a cada fila). MySQL solo admite **cursores explícitos**: hay que declararlos, abrirlos, leer de ellos y cerrarlos. A diferencia de Oracle, **no existe el cursor implícito** `FOR fila IN (SELECT ...) LOOP ... END LOOP;` — todo el recorrido hay que montarlo a mano con `LOOP` + `FETCH` + un `HANDLER` que detecte el final.

### Los cuatro pasos de un cursor

| Paso | Instrucción | Qué hace |
|---|---|---|
| 1. Declarar | `DECLARE cur CURSOR FOR SELECT ...;` | Asocia el cursor a una consulta (no la ejecuta todavía). |
| 2. Abrir | `OPEN cur;` | Ejecuta la consulta y posiciona el cursor antes de la primera fila. |
| 3. Leer | `FETCH cur INTO var1, var2, ...;` | Copia la fila actual en variables y avanza a la siguiente. |
| 4. Cerrar | `CLOSE cur;` | Libera los recursos asociados al cursor. |

El problema es saber **cuándo parar**: cuando un `FETCH` ya no encuentra más filas, lanza la condición `NOT FOUND`. Para no abortar el procedimiento con un error, se declara un `HANDLER` que la capture:

```sql
DECLARE v_fin INT DEFAULT FALSE;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_fin = TRUE;
```

`CONTINUE HANDLER` significa "cuando pase esto, no abortes: sigue ejecutando desde la siguiente instrucción" (en este caso, se usa para poner `v_fin` a verdadero y poder salir del bucle con `LEAVE` de forma controlada). Este es exactamente el patrón ya usado en el procedimiento `listar_prestamos_activos` del ejercicio de la biblioteca (Examen 2025) — este documento lo generaliza.

### Ejemplo completo

```sql
DELIMITER //
CREATE PROCEDURE listar_alumnos_ciclo(IN p_codciclo VARCHAR(10))
BEGIN
    DECLARE v_nia INT;
    DECLARE v_nombre VARCHAR(60);
    DECLARE v_fin INT DEFAULT FALSE;

    DECLARE cur_alumnos CURSOR FOR
        SELECT nia, nombre FROM Alumno WHERE codciclo = p_codciclo;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_fin = TRUE;

    OPEN cur_alumnos;
    bucle: LOOP
        FETCH cur_alumnos INTO v_nia, v_nombre;
        IF v_fin THEN
            LEAVE bucle;
        END IF;
        SELECT v_nia, v_nombre;  -- aquí iría la lógica fila a fila
    END LOOP;
    CLOSE cur_alumnos;
END //
DELIMITER ;

CALL listar_alumnos_ciclo('DAM');
```

!!! warning "Posible error: \"Illegal mix of collations\""
    Al ejecutar `CALL listar_alumnos_ciclo('DAM')` puede aparecer este error:

    ```text
    Error: Illegal mix of collations (utf8mb4_spanish_ci,IMPLICIT) and
    (utf8mb4_0900_ai_ci,IMPLICIT) for operation '='
    ```

    **Causa:** la base de datos ACADEMIA se creó con `COLLATE utf8mb4_spanish_ci`, pero MySQL 8 (a diferencia de MariaDB) usa por defecto la colación `utf8mb4_0900_ai_ci` para cualquier sesión nueva. El parámetro `p_codciclo` del procedimiento hereda esa colación de la sesión en el momento de crearse (no la de la tabla), así que al compararlo con la columna `codciclo` (que sí tiene la colación de ACADEMIA) MySQL se encuentra con dos colaciones distintas y no sabe cuál usar.

    **Solución (recomendada): declarar el parámetro con la misma colación que la tabla**

    ```sql
    DELIMITER //
    CREATE PROCEDURE listar_alumnos_ciclo(
        IN p_codciclo VARCHAR(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_spanish_ci
    )
    BEGIN
        ...
    END //
    DELIMITER ;
    ```

    **Alternativa:** forzar la colación directamente en la comparación, sin tocar el parámetro:

    ```sql
    SELECT nia, nombre FROM Alumno WHERE codciclo = p_codciclo COLLATE utf8mb4_spanish_ci;
    ```

    La primera opción es mejor si el parámetro se usa en varias consultas dentro del mismo procedimiento, porque arregla el problema una sola vez en la cabecera; la segunda es más rápida de aplicar si solo hay una comparación.

!!! warning "Orden obligatorio dentro del bloque"
    En MySQL, todas las `DECLARE` de cursores y de `HANDLER` deben ir **después** de las `DECLARE` de variables normales, pero **antes** de cualquier otra instrucción del bloque. Si se declara el cursor antes de sus variables, o el `HANDLER` antes del cursor que debe vigilar, el servidor da un error de sintaxis.

---

## Consultar y eliminar rutinas

```sql
SHOW PROCEDURE STATUS WHERE Db = 'academia';
SHOW FUNCTION STATUS WHERE Db = 'academia';
SHOW CREATE PROCEDURE insertar_nota;

DROP PROCEDURE IF EXISTS insertar_nota;
DROP FUNCTION IF EXISTS nota_media_alumno;
```

---

## Resumen

| Concepto | MySQL/MariaDB | Oracle PL/SQL |
|---|---|---|
| Delimitador de cuerpo | `DELIMITER //` ... `END //` | `/` tras el `END;` |
| Parámetros de función | Solo `IN` | `IN`, `OUT`, `IN OUT` |
| Bucle con rango numérico | `WHILE` (universal) / `FOR` (solo MariaDB) | `FOR i IN 1..n LOOP` |
| Cursor | Solo explícito | Explícito e implícito (`FOR r IN (SELECT...) LOOP`) |
| Filas afectadas | `ROW_COUNT()` | `SQL%ROWCOUNT` |
| Salida de depuración | Sin equivalente directo (`SELECT` o tabla auxiliar) | `DBMS_OUTPUT.PUT_LINE(...)` |
