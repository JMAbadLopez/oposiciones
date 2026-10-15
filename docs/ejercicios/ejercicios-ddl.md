# Ejercicios de examen: DDL (definición de bases de datos)

!!! note "Sobre estos ejercicios"
    Los ejercicios de esta página proceden de exámenes reales de las pruebas de acceso al cuerpo de profesores de Enseñanza Secundaria, especialidad Informática, convocadas por la Generalitat Valenciana (años 2021 y 2025). Se han transcrito el enunciado y los datos de partida; las soluciones son propias, verificadas ejecutándolas sobre MySQL/MariaDB con datos de prueba, y están ocultas en un desplegable para poder usarlas como ejercicio antes de consultar la solución.

---

## Examen 2021 (Modelo B, Ejercicio 2)

Un autónomo se dedica, a través de una franquicia, a distribuir cápsulas de café. El cliente presenta una hoja de cálculo desnormalizada, de la que se muestra una porción representativa:

| Client | DNI Client | Name | Phone | Address | Flavor | Unit Price | Amount | Short | Long | ExtraLarge | Intensity | Order | Order date | Sent |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 11111111-H | Antonio García | 961542145 | C Colón 16, Valencia | ExtraForte | 0,5 | 10 | NO | SÍ | SÍ | 12 | 1 | 01/08/2018 | SÍ |
| 1 | 11111111-H | Antonio García | 961542145 | C Colón 16, Valencia | Velluto | 0,45 | 5 | SÍ | SÍ | NO | 8 | 1 | 01/08/2018 | SÍ |
| 1 | 11111111-H | Antonio García | 961542145 | Av del Puerto 25, Valencia | Velluto | 0,48 | 5 | SÍ | NO | NO | 8 | 2 | 01/12/2016 | SÍ |
| 1 | 11111111-H | Antonio García | 961542145 | Av del Puerto 25, Valencia | Velluto | 0,4 | 20 | SÍ | NO | NO | 8 | 3 | 01/06/2016 | SÍ |
| 2 | 22222222-J | María Vicente | 961423544 | C Castellón 13, Alicante | Expresso | 0,5 | 50 | NO | NO | NO | 7 | 4 | 12/12/2020 | NO |

Los campos "Sí"/"No" no admiten otros valores. *Short*, *Long* y *ExtraLarge* indican la sugerencia de consumo de cada cápsula. *Sent* indica si se ha entregado el pedido.

Tareas (basadas en sintaxis SQL estándar):

- **a.** Introducir brevemente los conceptos aplicados, contextualizando en el temario oficial y en el ámbito de la formación profesional.
- **b.** Proponer una estructura de tablas a partir de la hoja de cálculo, hasta llegar a un conjunto de relaciones en tercera forma normal.
- **c.** Generar las consultas de creación de tablas **sin índices ni claves ni restricciones**.
- **d.** Generar las consultas de creación de claves, así como los índices especificados, indicando el SGBD utilizado.
- **e.** Generar la consulta que obtiene un listado ordenado por apellido del cliente, con la cantidad de pedidos que ha realizado y el total facturado por ellos.
- **f.** Proponer una optimización de las columnas *Short*, *Long* y *ExtraLarge* representándolas en una sola columna mediante operadores *bitwise*, y generar la consulta que obtiene la cantidad de cápsulas vendidas válidas tanto para *Long* como para *ExtraLarge*.

??? success "Solución: normalización (apartado b)"
    La hoja de cálculo mezcla varias entidades distintas en las mismas columnas, con dependencias funcionales diferentes:

    - El **cliente** (DNI, nombre, teléfono) se repite idéntico en todas las filas de un mismo `Client`: depende solo de `Client`.
    - La **dirección de entrega** cambia entre pedidos de un mismo cliente (ver filas 1-2 frente a 3-4): depende del **pedido**, no del cliente.
    - El **sabor** (`Flavor`) tiene su propio precio y, en esta propuesta, su formato (*short/long/extralarge*) e intensidad: se trata como una entidad aparte.
    - `Amount` (la cantidad de cápsulas) depende de la combinación pedido + sabor: es un atributo de la línea de pedido.

    Esquema propuesto (claves subrayadas):

    - **Clientes** (<u>id_cliente</u>, dni, nombre, apellidos, telefono)
    - **Pedidos** (<u>id_pedido</u>, id_cliente, direccion_entrega, fecha_pedido, enviado)
    - **Sabores** (<u>id_sabor</u>, nombre, precio_unitario)
    - **Lineas_Pedido** (<u>id_pedido, id_sabor</u>, cantidad, formato, intensidad)

    Con esta estructura ningún atributo no clave depende de una parte de una clave compuesta ni de otro atributo no clave: está en 3FN.

    !!! warning "Ojo con la atomicidad: Name no es un único atributo"
        La columna `Name` de la hoja de cálculo ("Antonio García", "María Vicente") en realidad mezcla **dos** datos distintos: el nombre de pila y los apellidos. Guardarlos juntos en una sola columna viola la **1ª Forma Normal** (que exige valores atómicos, no compuestos), y además obliga a usar trucos de manipulación de texto (como extraer la última palabra) cada vez que se necesita ordenar o filtrar solo por apellido — algo frágil en cuanto aparece un apellido compuesto o alguien con varios nombres. Por eso el esquema separa `nombre` y `apellidos` en dos columnas de `Clientes`, en vez de una sola columna `nombre` combinada. El apartado e) de este ejercicio (ordenar por apellido) es precisamente el que pone de manifiesto por qué conviene separarlos desde el diseño.

??? success "Solución: creación de tablas sin claves ni restricciones (apartado c)"
    ```sql
    CREATE TABLE Clientes (
        id_cliente INT,
        dni VARCHAR(10),
        nombre VARCHAR(50),
        apellidos VARCHAR(100),
        telefono VARCHAR(15)
    );

    CREATE TABLE Sabores (
        id_sabor INT,
        nombre VARCHAR(30),
        precio_unitario DECIMAL(5,2)
    );

    CREATE TABLE Pedidos (
        id_pedido INT,
        id_cliente INT,
        direccion_entrega VARCHAR(100),
        fecha_pedido DATE,
        enviado VARCHAR(2)
    );

    CREATE TABLE Lineas_Pedido (
        id_pedido INT,
        id_sabor INT,
        cantidad INT,
        formato TINYINT,
        intensidad INT
    );
    ```

??? success "Solución: claves e índices (apartado d)"
    SGBD utilizado: **MySQL / MariaDB**. Las claves se añaden con `ALTER TABLE` (tal como pide el enunciado) después de haber creado las tablas:

    ```sql
    ALTER TABLE Clientes
        ADD CONSTRAINT pk_clientes PRIMARY KEY (id_cliente),
        ADD CONSTRAINT uq_clientes_dni UNIQUE (dni);

    ALTER TABLE Sabores
        ADD CONSTRAINT pk_sabores PRIMARY KEY (id_sabor),
        ADD CONSTRAINT uq_sabores_nombre UNIQUE (nombre);

    ALTER TABLE Pedidos
        ADD CONSTRAINT pk_pedidos PRIMARY KEY (id_pedido),
        ADD CONSTRAINT fk_pedidos_cliente FOREIGN KEY (id_cliente) REFERENCES Clientes(id_cliente),
        ADD CONSTRAINT chk_enviado CHECK (enviado IN ('SI','NO'));

    CREATE INDEX idx_pedidos_cliente ON Pedidos(id_cliente);

    ALTER TABLE Lineas_Pedido
        ADD CONSTRAINT pk_lineas_pedido PRIMARY KEY (id_pedido, id_sabor),
        ADD CONSTRAINT fk_lineas_pedido FOREIGN KEY (id_pedido) REFERENCES Pedidos(id_pedido),
        ADD CONSTRAINT fk_lineas_sabor FOREIGN KEY (id_sabor) REFERENCES Sabores(id_sabor),
        ADD CONSTRAINT chk_formato CHECK (formato BETWEEN 0 AND 7);

    CREATE INDEX idx_lineas_sabor ON Lineas_Pedido(id_sabor);
    ```

    La clave primaria de `Pedidos` y de `Sabores` crea automáticamente un índice; se añaden además índices explícitos sobre las claves ajenas (`id_cliente` en `Pedidos`, `id_sabor` en `Lineas_Pedido`), ya que MySQL/MariaDB no los crea de forma automática al declarar una `FOREIGN KEY` salvo que ya exista uno adecuado.

!!! tip "Propuesta de optimización con operadores bitwise (apartado f)"
    En lugar de tres columnas `Short`, `Long`, `ExtraLarge` (cada una "SÍ"/"NO"), se puede representar la combinación con una sola columna `formato TINYINT`, usando un bit por cada opción:

    | Bit | Valor | Significa |
    |---|---|---|
    | 0 | 1 | Short |
    | 1 | 2 | Long |
    | 2 | 4 | ExtraLarge |

    Una cápsula válida para *Short* y *Long* a la vez tendría `formato = 1 + 2 = 3`. Para comprobar si un bit está activado se usa el operador `&` (AND bit a bit): `(formato & 2) = 2` es verdadero si el bit de *Long* está activo, con independencia de los demás bits.

    La ventaja es compactar tres columnas en una sola, consultable con operadores bitwise en vez de tres condiciones `AND` separadas; el inconveniente es que la columna deja de ser autoexplicativa y hay que documentar el significado de cada bit.

    ```sql
    -- Cantidad de capsulas vendidas validas tanto para Long como para ExtraLarge
    SELECT SUM(LP.cantidad) AS capsulas_long_y_extralarge
    FROM Lineas_Pedido LP
    WHERE (LP.formato & 2) = 2
      AND (LP.formato & 4) = 4;
    ```

---

## Examen 2025 (Ejercicio 2: Problema BD)

Eres docente en un centro educativo y preparas una práctica sobre una base de datos relacional de biblioteca: lectores, categorías de libros (con subcategorías), autores (con posible autor inspirador), libros y préstamos. La base de datos debe controlar la disponibilidad de libros (`cantidad_disponible`), permitir consultas sobre libros/autores/categorías/préstamos, garantizar que no se presten libros sin copias disponibles y generar informes (libros más prestados, usuarios con préstamos vencidos).

El enunciado ofrece el DDL en Oracle, indicando que se puede adaptar a MySQL con los cambios `NUMBER → INT`, `VARCHAR2 → VARCHAR` y `CURRENT_TIMESTAMP`/`SYSDATE → DATETIME` o `TIMESTAMP`/`DATE DEFAULT (CURRENT_DATE)`.

Dos de las ocho tareas del ejercicio son estrictamente de definición de datos (DDL): crear la propia estructura de tablas, y crear la tabla y el disparador de auditoría del apartado 8. (Las consultas y el procedimiento con cursor, que son DML, se recogen en el documento de ejercicios de manipulación).

### Estructura de la base de datos (adaptada a MySQL/MariaDB)

??? success "Solución: DDL adaptado a MySQL (lectores, categorías, autores, libros, préstamos)"
    ```sql
    CREATE TABLE lectores (
        id_lector INT PRIMARY KEY,
        nombre VARCHAR(100) NOT NULL,
        email VARCHAR(100) UNIQUE NOT NULL,
        fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE categorias (
        id_categoria INT PRIMARY KEY,
        nombre VARCHAR(100) NOT NULL,
        id_categoria_padre INT NULL,
        CONSTRAINT fk_categoria_padre FOREIGN KEY (id_categoria_padre)
            REFERENCES categorias(id_categoria) ON DELETE SET NULL
    );

    CREATE TABLE autores (
        id_autor INT PRIMARY KEY,
        nombre VARCHAR(100) NOT NULL,
        id_autor_inspirador INT NULL,
        CONSTRAINT fk_autor_inspirador FOREIGN KEY (id_autor_inspirador)
            REFERENCES autores(id_autor) ON DELETE SET NULL
    );

    CREATE TABLE libros (
        id_libro INT PRIMARY KEY,
        titulo VARCHAR(255) NOT NULL,
        id_categoria INT NULL,
        id_autor INT NOT NULL,
        anio_publicacion INT NOT NULL,
        cantidad_disponible INT DEFAULT 1,
        CONSTRAINT fk_libro_categoria FOREIGN KEY (id_categoria) REFERENCES categorias(id_categoria) ON DELETE SET NULL,
        CONSTRAINT fk_libro_autor FOREIGN KEY (id_autor) REFERENCES autores(id_autor) ON DELETE CASCADE,
        CONSTRAINT chk_anio_publicacion CHECK (anio_publicacion >= 1400),
        CONSTRAINT chk_cantidad_disponible CHECK (cantidad_disponible >= 0)
    );

    CREATE TABLE prestamos (
        id_prestamo INT PRIMARY KEY,
        id_lector INT NOT NULL,
        id_libro INT NOT NULL,
        fecha_prestamo DATE DEFAULT (CURRENT_DATE),
        fecha_devolucion DATE NULL,
        CONSTRAINT fk_prestamo_lector FOREIGN KEY (id_lector) REFERENCES lectores(id_lector) ON DELETE CASCADE,
        CONSTRAINT fk_prestamo_libro FOREIGN KEY (id_libro) REFERENCES libros(id_libro) ON DELETE CASCADE,
        CONSTRAINT chk_fechas CHECK (fecha_devolucion IS NULL OR fecha_devolucion >= fecha_prestamo)
    );
    ```

    !!! warning "Un detalle a corregir del enunciado original"
        El DDL de `libros` declaraba `id_categoria NUMBER NOT NULL` pero su clave ajena usa `ON DELETE SET NULL`: si la columna no admite NULL, borrar una categoría con libros asociados provocaría un error de integridad en lugar de poner la columna a NULL. Para que la regla `ON DELETE SET NULL` funcione, `id_categoria` debe admitir NULL (`id_categoria INT NULL`), como se ha corregido arriba. Es un buen ejemplo para comentar en clase: una clave ajena `ON DELETE SET NULL` exige que la columna sea *nullable*.

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

## Preguntas tipo test relacionadas con DDL

Ambos exámenes incluyen, además de los ejercicios prácticos anteriores, un bloque inicial de 25 (2021) o 150 (2025) preguntas tipo test. Estas son las que tratan temas de definición de bases de datos.

### Examen 2021 (Modelo A, pregunta 25)

¿Cuál es la diferencia entre clave primaria y clave única?

- **a)** La clave única permite múltiples valores nulos y la clave primaria no.
- **b)** La clave primaria permite un valor nulo mientras que la clave única no permite valor nulo.
- **c)** La clave primaria no permite valor nulo mientras que la clave única permite un valor nulo.
- **d)** Ninguna de las anteriores.

??? success "Solución: a)"
    Una clave primaria (`PRIMARY KEY`) no admite valores `NULL` en ninguna de sus columnas y solo puede haber una por tabla. Una clave única (`UNIQUE`) sí admite `NULL` (y, en MySQL/MariaDB, puede admitir varios `NULL`, ya que dos `NULL` nunca se consideran iguales entre sí a efectos de la restricción de unicidad); una tabla puede tener varias claves únicas. Es un contenido típico del tema de DDL: toda clave primaria es, de hecho, un caso particular de clave única con la restricción adicional de `NOT NULL`.
