# Ejercicios de examen: DML (manipulación de datos)

!!! note "Sobre estos ejercicios"
    Los ejercicios de esta página proceden de exámenes reales de las pruebas de acceso al cuerpo de profesores de Enseñanza Secundaria, especialidad Informática, convocadas por la Generalitat Valenciana (años 2021 y 2025). Son la continuación de los ejercicios de DDL de los mismos exámenes: aquí se recogen las consultas (`SELECT`), trabajando ya sobre las tablas creadas en el documento de DDL. El procedimiento con cursor del mismo ejercicio de 2025 se recoge en el documento de ejercicios de procedimientos, funciones y disparadores. Las soluciones son propias, verificadas ejecutándolas sobre MySQL/MariaDB con datos de prueba, y están ocultas en un desplegable para poder usarlas como ejercicio antes de consultar la solución.

---

## Examen 2021 (Modelo B, Ejercicio 2): distribuidora de cápsulas de café

Se parte del esquema normalizado en el documento de DDL:

- **Clientes** (<u>id_cliente</u>, dni, nombre, apellidos, telefono)
- **Pedidos** (<u>id_pedido</u>, id_cliente, direccion_entrega, fecha_pedido, enviado)
- **Sabores** (<u>id_sabor</u>, nombre, precio_unitario)
- **Lineas_Pedido** (<u>id_pedido, id_sabor</u>, cantidad, formato, intensidad)

### Listado de clientes ordenado por apellido, con pedidos y total facturado

Enunciado: obtener un listado ordenado por apellido del cliente, con la cantidad de pedidos que ha realizado y el total facturado por ellos.

??? success "Solución"
    ```sql
    SELECT
        C.nombre,
        C.apellidos,
        COUNT(DISTINCT P.id_pedido) AS num_pedidos,
        SUM(LP.cantidad * S.precio_unitario) AS total_facturado
    FROM Clientes C
    JOIN Pedidos P ON P.id_cliente = C.id_cliente
    JOIN Lineas_Pedido LP ON LP.id_pedido = P.id_pedido
    JOIN Sabores S ON S.id_sabor = LP.id_sabor
    GROUP BY C.id_cliente, C.nombre, C.apellidos
    ORDER BY C.apellidos;
    ```

    Resultado real (con los datos de ejemplo):

    | nombre | apellidos | num_pedidos | total_facturado |
    |---|---|---|---|
    | Antonio | García | 3 | 18.50 |
    | María | Vicente | 1 | 25.00 |

    Notas:

    - `COUNT(DISTINCT P.id_pedido)` es necesario porque el `JOIN` con `Lineas_Pedido` genera una fila por cada línea de pedido: si se contara `P.id_pedido` sin `DISTINCT`, un pedido con varias líneas se contaría varias veces.
    - El importe de cada línea es `cantidad * precio_unitario`; `SUM` lo agrega para todas las líneas de todos los pedidos de cada cliente.
    - Como el esquema separa `nombre` y `apellidos` en dos columnas (ver la nota de normalización en el documento de DDL), ordenar "por apellido" es tan sencillo como `ORDER BY C.apellidos`, sin necesidad de ningún truco de manipulación de texto para extraer el apellido de un campo combinado.

---

## Examen 2025 (Ejercicio 2: Problema BD): biblioteca

Se parte del esquema creado en el documento de DDL (`lectores`, `categorias`, `autores`, `libros`, `prestamos`), con `cantidad_disponible` en `libros` y `fecha_prestamo`/`fecha_devolucion` en `prestamos` (`fecha_devolucion` a `NULL` mientras el préstamo sigue activo). El plazo de devolución considerado es de 15 días.

### 1) Los 5 libros más prestados

??? success "Solución"
    ```sql
    SELECT L.titulo, COUNT(*) AS veces_prestado
    FROM prestamos P
    JOIN libros L ON L.id_libro = P.id_libro
    GROUP BY L.id_libro, L.titulo
    ORDER BY veces_prestado DESC
    LIMIT 5;
    ```

    `GROUP BY` agrupa los préstamos por libro, `COUNT(*)` cuenta cuántas filas (préstamos) tiene cada grupo, y `ORDER BY ... DESC LIMIT 5` se queda con los cinco libros más prestados. Si hay menos de cinco libros distintos prestados, simplemente se devuelven todos.

### 2) Lectores con préstamos vencidos

Enunciado: nombre, correo electrónico, título del libro y fecha del préstamo de las personas con préstamos vencidos (plazo de devolución de 15 días, sin devolución registrada).

??? success "Solución"
    ```sql
    SELECT R.nombre, R.email, L.titulo, P.fecha_prestamo
    FROM prestamos P
    JOIN lectores R ON R.id_lector = P.id_lector
    JOIN libros L ON L.id_libro = P.id_libro
    WHERE P.fecha_devolucion IS NULL
      AND P.fecha_prestamo < CURDATE() - INTERVAL 15 DAY;
    ```

    Un préstamo está "vencido" cuando no se ha devuelto (`fecha_devolucion IS NULL`) y, además, han pasado más de 15 días desde que se prestó, lo que equivale a que la fecha de préstamo sea anterior a "hoy menos 15 días".

### 3) Lector con más préstamos activos sin devolver

Enunciado: nombre, correo electrónico, ID y cantidad de libros prestados sin devolver del lector con más préstamos activos, limitando el resultado a un solo registro.

??? success "Solución"
    ```sql
    SELECT R.id_lector, R.nombre, R.email, COUNT(*) AS prestamos_activos
    FROM prestamos P
    JOIN lectores R ON R.id_lector = P.id_lector
    WHERE P.fecha_devolucion IS NULL
    GROUP BY R.id_lector, R.nombre, R.email
    ORDER BY prestamos_activos DESC
    LIMIT 1;
    ```

    Mismo patrón que el ejercicio anterior (agrupar y contar), combinado con `LIMIT 1` para quedarse solo con el lector que encabeza la clasificación. Si hubiera un empate, `LIMIT 1` devolvería arbitrariamente uno de los lectores empatados: si el enunciado exigiera mostrar a todos los empatados, habría que usar en su lugar una subconsulta correlacionada comparando con el máximo, en vez de `LIMIT`.

### 4) Lectores que nunca han devuelto un libro con retraso

Enunciado: nombres de los lectores que nunca han devuelto un libro con retraso (más de 15 días después de la fecha del préstamo).

??? success "Solución"
    ```sql
    SELECT R.nombre
    FROM lectores R
    WHERE NOT EXISTS (
        SELECT * FROM prestamos P
        WHERE P.id_lector = R.id_lector
          AND P.fecha_devolucion IS NOT NULL
          AND DATEDIFF(P.fecha_devolucion, P.fecha_prestamo) > 15
    );
    ```

    Es el patrón de doble negación típico de la cuantificación universal en SQL: "lectores para los que **no existe** ningún préstamo devuelto con retraso" equivale a "lectores que **nunca** han devuelto con retraso". `DATEDIFF(fecha_devolucion, fecha_prestamo)` calcula los días transcurridos entre el préstamo y la devolución.

### 5) Todos los libros que han sido prestados al menos una vez

??? success "Solución"
    ```sql
    SELECT DISTINCT L.titulo
    FROM libros L
    JOIN prestamos P ON P.id_libro = L.id_libro;
    ```

    El `JOIN` con `prestamos` descarta automáticamente los libros que nunca aparecen en esa tabla (los que no se han prestado nunca); `DISTINCT` evita repetir el título de un libro que se haya prestado varias veces.

!!! note "Procedimiento con cursor de este ejercicio"
    El apartado 7 de este mismo ejercicio (un procedimiento con cursor sobre `prestamos`) se recoge en el documento de ejercicios de **procedimientos, funciones y disparadores**, junto con el disparador del apartado 8 y las preguntas tipo test relacionadas.

---

## Preguntas tipo test relacionadas con DML

Ambos exámenes incluyen, además de los ejercicios prácticos anteriores, un bloque inicial de 25 (2021) o 150 (2025) preguntas tipo test. Estas son las que tratan sintaxis o conceptos de manipulación de datos.

### Examen 2021 (Modelo A, pregunta 1)

En MySQL las actualizaciones se llevan a cabo mediante la sentencia:

- **a)** `UPDATE nombre_tabla SET columna_1 = expresion1, columna_2 = expresion2, columna_n = expresion WHERE condicion`
- **b)** `SELECT nombre_tabla SET columna_1 = expresion1, columna_2 = expresion2, columna_n = expresion WHERE condicion`
- **c)** `SHOW nombre_tabla SET columna_1 = expresion1, columna_2 = expresion2, columna_n = expresion WHERE condicion`
- **d)** `INSERT nombre_tabla SET columna_1 = expresion1, columna_2 = expresion2, columna_n = expresion WHERE condicion`

??? success "Solución: a)"
    Es la sintaxis estándar de `UPDATE` vista en el tema de manipulación: `UPDATE tabla SET columna = valor, ... WHERE condicion`. Las otras tres opciones combinan la cláusula `SET ... WHERE` (propia de `UPDATE`) con el nombre de otra instrucción (`SELECT`, `SHOW`, `INSERT`), que no admite esa sintaxis.

### Examen 2025 (preguntas 96-97)

Estas dos preguntas comparten el mismo esquema de base de datos (las preguntas 98-100 del mismo bloque, sobre cursores, procedimientos y gestión de errores, se recogen en el documento de ejercicios de procedimientos, funciones y disparadores):

```
AUTORES(id_autor, login, password, correo, url)
NOTICIAS(id_noticia, id_autor, titulo, fecha_pub, contenido, id_link)
COMENTARIOS(id_comentario, id_noticia, id_autor, fecha_comen, contenido)
ENLACES(id_link, link_url, url_status)
```

**96.** ¿Cuál es la consulta SQL correcta para listar los títulos de noticias y el contenido de los comentarios hechos por el autor con ID 5, ordenados por fecha descendente?

- **a)** `SELECT N.titulo, C.contenido FROM NOTICIAS N JOIN COMENTARIOS C ON C.id_autor = 5 ORDER BY N.fecha_pub DESC;`
- **b)** `SELECT N.titulo, C.contenido FROM COMENTARIOS C WHERE C.id_autor = 5 ORDER BY C.fecha_comen;`
- **c)** `SELECT N.titulo, C.contenido FROM COMENTARIOS C JOIN NOTICIAS N ON C.id_noticia = N.id_noticia WHERE C.id_autor = 5 ORDER BY C.fecha_comen DESC;`
- **d)** `SELECT N.titulo, C.contenido FROM COMENTARIOS C JOIN NOTICIAS N ON C.id = N.id WHERE C.id_autor = 5;`

??? success "Solución: c)"
    La a) no tiene una condición de unión (`ON`) correcta entre `NOTICIAS` y `COMENTARIOS` (el `ON` compara `id_autor` con una constante en lugar de relacionar las dos tablas, lo que produciría un producto cartesiano filtrado incorrectamente). La b) no une ambas tablas, así que `N.titulo` no estaría definido para cada fila. La d) usa `C.id` y `N.id`, columnas que no existen en este esquema (las claves se llaman `id_comentario`, `id_noticia`, etc.). La c) une correctamente `COMENTARIOS` con `NOTICIAS` por `id_noticia`, filtra por el autor del comentario y ordena por la fecha del comentario en orden descendente, tal como pide el enunciado.

**97.** ¿Cuál es la consulta SQL correcta para actualizar `url_status` a 'Válida' en la tabla `ENLACES` para URLs que comienzan con `'https://'`?

- **a)** `UPDATE ENLACES SET link_url = 'Válida' WHERE url_status LIKE 'https/%';`
- **b)** `ALTER ENLACES SET url_status = 'Válida' WHERE link_url LIKE 'https://%';`
- **c)** `UPDATE ENLACES SET url_status = 'Válida' WHERE link_url LIKE 'https://%';`
- **d)** `UPDATE ENLACES SET url_status = 'Válida' WHERE link_url = 'https://';`

??? success "Solución: c)"
    La a) actualiza la columna equivocada (`link_url` en vez de `url_status`) y compara `url_status` con un patrón que no tiene sentido para esa columna. La b) usa `ALTER`, que es DDL (modifica la estructura de una tabla), no DML: para cambiar valores de filas hace falta `UPDATE`. La d) compara con el valor exacto `'https://'` (`=`) en lugar de con un patrón, por lo que no capturaría ninguna URL real que empiece por ese prefijo. La c) es la única que usa `UPDATE` con `SET` sobre la columna correcta (`url_status`) y filtra con `LIKE 'https://%'`, el patrón correcto para "empieza por https://".
