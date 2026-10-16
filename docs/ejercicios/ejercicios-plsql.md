# Ejercicios de examen: Procedimientos, funciones y disparadores

!!! note "Sobre estos ejercicios"
    Los ejercicios de esta página proceden de exámenes reales de las pruebas de acceso al cuerpo de profesores de Enseñanza Secundaria, especialidad Informática, convocadas por la Generalitat Valenciana (años 2021 y 2025). Son la continuación de los ejercicios de DDL y DML del mismo examen de 2025: aquí se recogen las tareas que piden escribir un procedimiento con cursor y un disparador, trabajando sobre las tablas creadas en el documento de DDL. Las soluciones son propias, verificadas ejecutándolas sobre MySQL/MariaDB con datos de prueba, y están ocultas en un desplegable para poder usarlas como ejercicio antes de consultar la solución.

---

## Examen 2025 (Ejercicio 2: Problema BD): biblioteca

Se parte del esquema creado en el documento de DDL (`lectores`, `categorias`, `autores`, `libros`, `prestamos`), con `cantidad_disponible` en `libros` y `fecha_prestamo`/`fecha_devolucion` en `prestamos` (`fecha_devolucion` a `NULL` mientras el préstamo sigue activo).

### Apartado 7: procedimiento con cursor — préstamos activos ordenados por fecha

Enunciado: crear un procedimiento que, mediante un cursor, genere un listado de préstamos activos ordenados por fecha de préstamo, incluyendo el ID del préstamo, el nombre del lector, el título del libro y la fecha del préstamo; el procedimiento debe abrir el cursor, recorrerlo y mostrar los datos obtenidos.

??? success "Solución"
    ```sql
    DELIMITER //
    CREATE PROCEDURE listar_prestamos_activos()
    BEGIN
        DECLARE v_id_prestamo INT;
        DECLARE v_nombre VARCHAR(100);
        DECLARE v_titulo VARCHAR(255);
        DECLARE v_fecha DATE;
        DECLARE v_fin INT DEFAULT FALSE;

        DECLARE cur_prestamos CURSOR FOR
            SELECT P.id_prestamo, R.nombre, L.titulo, P.fecha_prestamo
            FROM prestamos P
            JOIN lectores R ON R.id_lector = P.id_lector
            JOIN libros L ON L.id_libro = P.id_libro
            WHERE P.fecha_devolucion IS NULL
            ORDER BY P.fecha_prestamo;

        DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_fin = TRUE;

        OPEN cur_prestamos;

        bucle: LOOP
            FETCH cur_prestamos INTO v_id_prestamo, v_nombre, v_titulo, v_fecha;
            IF v_fin THEN
                LEAVE bucle;
            END IF;
            SELECT v_id_prestamo, v_nombre, v_titulo, v_fecha;
        END LOOP;

        CLOSE cur_prestamos;
    END //
    DELIMITER ;

    CALL listar_prestamos_activos();
    ```

    Estructura de un cursor explícito en MySQL/MariaDB:

    1. `DECLARE ... CURSOR FOR <SELECT>` asocia el cursor a la consulta que se quiere recorrer (aquí, ya ordenada con `ORDER BY`).
    2. `DECLARE CONTINUE HANDLER FOR NOT FOUND ...` define qué hacer cuando un `FETCH` ya no encuentra más filas: en vez de lanzar un error, pone a `TRUE` una variable de control (`v_fin`).
    3. `OPEN` abre el cursor; cada `FETCH ... INTO ...` avanza una fila y copia sus columnas en variables locales.
    4. El bucle (`LOOP` / `LEAVE`) repite el `FETCH` hasta que `v_fin` se activa, momento en el que se sale del bucle con `LEAVE bucle`.
    5. `CLOSE` libera el cursor al terminar.

    `DELIMITER //` es necesario porque el cuerpo del procedimiento contiene punto y coma (`;`) propios, y hay que evitar que el cliente de MySQL interprete cada uno como el final de la instrucción `CREATE PROCEDURE`.

### Apartado 8: tabla y disparador de auditoría

Enunciado: crear un disparador (*trigger*) que audite los libros borrados o actualizados, registrando en una tabla de auditoría la fecha de la operación, el usuario del SGBD y el tipo de operación. El disparador debe ejecutarse solo ante `DELETE` o `UPDATE`.

??? success "Solución"
    ```sql
    CREATE TABLE auditoria_libros (
        id_auditoria INT AUTO_INCREMENT PRIMARY KEY,
        id_libro INT,
        operacion VARCHAR(10),
        usuario_sgbd VARCHAR(100),
        fecha_operacion DATETIME DEFAULT CURRENT_TIMESTAMP
    );

    DELIMITER //
    CREATE TRIGGER trg_libros_delete
    AFTER DELETE ON libros
    FOR EACH ROW
    BEGIN
        INSERT INTO auditoria_libros (id_libro, operacion, usuario_sgbd)
        VALUES (OLD.id_libro, 'DELETE', CURRENT_USER());
    END //

    CREATE TRIGGER trg_libros_update
    AFTER UPDATE ON libros
    FOR EACH ROW
    BEGIN
        INSERT INTO auditoria_libros (id_libro, operacion, usuario_sgbd)
        VALUES (OLD.id_libro, 'UPDATE', CURRENT_USER());
    END //
    DELIMITER ;
    ```

    En MySQL/MariaDB no existe un único disparador que capture a la vez `DELETE` y `UPDATE`: cada disparador se asocia a un evento y un momento concretos (`BEFORE`/`AFTER` + `INSERT`/`UPDATE`/`DELETE`), así que hacen falta dos disparadores independientes, uno por evento. Se usa `OLD.id_libro` porque en un `DELETE` solo existe la fila anterior a la operación (`OLD`); en un `UPDATE` también está disponible `NEW` con los valores nuevos, pero para esta auditoría basta con identificar qué libro se vio afectado.

    Comprobación (actualizando y borrando un libro de prueba):

    ```sql
    UPDATE libros SET cantidad_disponible = 2 WHERE id_libro = 3;
    DELETE FROM libros WHERE id_libro = 4;

    SELECT * FROM auditoria_libros;
    ```

    Resultado real:

    | id_auditoria | id_libro | operacion | usuario_sgbd | fecha_operacion |
    |---|---|---|---|---|
    | 1 | 3 | UPDATE | root@localhost | (fecha/hora actual) |
    | 2 | 4 | DELETE | root@localhost | (fecha/hora actual) |

---

## Preguntas tipo test relacionadas con procedimientos, funciones y disparadores

El examen de 2025 incluye, además del ejercicio práctico anterior, un bloque inicial de 150 preguntas tipo test. Estas tres preguntas comparten el mismo esquema de base de datos y tratan específicamente sobre cursores, procedimientos y gestión de errores:

```
AUTORES(id_autor, login, password, correo, url)
NOTICIAS(id_noticia, id_autor, titulo, fecha_pub, contenido, id_link)
COMENTARIOS(id_comentario, id_noticia, id_autor, fecha_comen, contenido)
ENLACES(id_link, link_url, url_status)
```

### Examen 2025 (pregunta 98)

¿Cuál es el fragmento de código SQL correcto para declarar un cursor que recorra todos los comentarios de la noticia con identificador 7?

- **a)** `DECLARE cur_comentarios CURSOR FOR SELECT id_comentario, contenido FROM COMENTARIOS WHERE id_autor = 7;`
- **b)** `DECLARE cur_comentarios CURSOR FOR SELECT id_comentario, contenido FROM NOTICIAS WHERE noticia = 7;`
- **c)** `DECLARE cur_comentarios CURSOR FOR SELECT id_comentario, contenido FROM COMENTARIOS ORDER BY noticia;`
- **d)** `DECLARE cur_comentarios CURSOR FOR SELECT id_comentario, contenido FROM COMENTARIOS WHERE id_noticia = 7;`

??? success "Solución: d)"
    La a) filtra por `id_autor` en vez de por la noticia. La b) consulta la tabla `NOTICIAS` (que no tiene columna `contenido` de comentario) y usa una columna `noticia` que no existe. La c) no filtra por la noticia 7 en absoluto, y además usa una columna `noticia` inexistente. La d) selecciona de `COMENTARIOS` filtrando correctamente por `id_noticia = 7`, la columna real que relaciona un comentario con su noticia.

### Examen 2025 (pregunta 99)

¿Cuál es el procedimiento SQL (MySQL) correcto para contar los comentarios por noticia y mostrar el título de la noticia junto con el total de comentarios?

- **a)** `CREATE PROCEDURE contar_comentarios_por_noticia() BEGIN SELECT N.titulo, COUNT(C.id_comentario) AS total FROM NOTICIAS N LEFT JOIN COMENTARIOS C ON N.id_noticia = C.id_noticia GROUP BY N.id_noticia, N.titulo; END;`
- **b)** `CREATE PROCEDURE contar_comentarios_por_noticia() BEGIN SELECT N.id_titulo, COUNT(C.id_comentario) AS total FROM NOTICIAS N LEFT JOIN COMENTARIOS C ON N.id_noticia = C.id_noticia GROUP BY N.id_noticia, N.titulo; END;`
- **c)** `CREATE PROCEDURE contar_comentarios_por_noticia() BEGIN SELECT N.titulo, SUM(C.id_comentario) AS total FROM NOTICIAS N JOIN COMENTARIOS C ON N.id_noticia = C.id_noticia; END;`
- **d)** `CREATE PROCEDURE contar_comentarios_por_noticia() BEGIN SELECT N.id_titulo, COUNT(C.id_comentario) AS total FROM NOTICIAS N JOIN COMENTARIOS C ON N.id_noticia = C.id_autor GROUP BY N.id; END;`

??? success "Solución: a)"
    La b) y la d) seleccionan `N.id_titulo`, una columna que no existe (la columna se llama `titulo`). La c) usa `SUM(C.id_comentario)` en vez de `COUNT(C.id_comentario)` (sumar identificadores no tiene sentido como "total de comentarios") y, además, usa `JOIN` en vez de `LEFT JOIN`, con lo que las noticias sin comentarios desaparecerían del resultado. La d) además une mal las tablas (`C.id_autor` en vez de `C.id_noticia`). La a) es correcta: `LEFT JOIN` conserva las noticias sin comentarios (mostrando `0` gracias a `COUNT`, que no cuenta los `NULL` que deja el `LEFT JOIN`), y agrupa por noticia para obtener un total por cada una.

### Examen 2025 (pregunta 100)

¿Cuál es el fragmento de código SQL (MySQL) correcto para gestionar el fin de un cursor en un procedimiento que recorre la tabla `ENLACES`?

- **a)** `DECLARE HANDLER FOR NOT FOUND SET done = FALSE;`
- **b)** `DECLARE CONTINUE HANDLER FOR SQLERROR SET done = TRUE;`
- **c)** `DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;`
- **d)** `DECLARE EXIT HANDLER FOR SQLSTATE '02000' SET done = FALSE;`

??? success "Solución: c)"
    La a) está incompleta: a un `DECLARE HANDLER` le falta indicar el tipo de manejador (`CONTINUE` o `EXIT`), es sintácticamente inválida en MySQL. La b) usa la condición `SQLERROR`, que no existe en MySQL (la condición correcta para errores genéricos sería `SQLEXCEPTION`), y además pone `done = TRUE`, lo que detendría el procedimiento ante cualquier error, no solo al agotar el cursor. La d) usa `EXIT HANDLER`, que abandonaría el procedimiento entero en vez de solo salir del bucle de lectura, y pone `done = FALSE` (al revés de lo necesario). La c) es el patrón estándar visto en el ejercicio del procedimiento con cursor: `CONTINUE HANDLER FOR NOT FOUND` captura el momento en que un `FETCH` ya no encuentra más filas (el código SQLSTATE `02000`, "no data") y permite seguir ejecutando el procedimiento (en vez de abortarlo) para poder salir del bucle de forma controlada.
