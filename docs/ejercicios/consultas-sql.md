# Ejercicios de examen: consultas SQL

!!! note "Sobre estos ejercicios"
    Los ejercicios de esta página proceden de exámenes reales de las pruebas de acceso al cuerpo de profesores de Enseñanza Secundaria, especialidad Informática, convocadas por la Generalitat Valenciana (años 2004, 2005 y 2009). Se han transcrito literalmente el enunciado y el esquema de cada base de datos; las soluciones son propias, verificadas ejecutándolas sobre MySQL/MariaDB con datos de prueba, y están ocultas en un desplegable para poder usarlas como ejercicio antes de consultar la solución.

---

## Examen 2004

De la base de datos necesaria para gestionar una pizzería se extraen las siguientes tablas:

- **PIZZAS** (<u>NPizza</u>, Precio): almacena para cada pizza su nombre y su precio.
- **ELABORACIÓN** (<u>NPizza, Ingrediente</u>, Cantidad): almacena para cada pizza su nombre, los nombres de sus ingredientes y la cantidad (en gramos) de cada uno de ellos utilizada en su elaboración. (Se supone que la masa se considera un ingrediente más).

### 1) Pizzas de más de 15 € con menos de 4 ingredientes

Obtener los nombres de las pizzas de más de 15 euros que lleven menos de 4 ingredientes.

??? success "Solución"
    ```sql
    SELECT P.NPizza
    FROM PIZZAS P
    WHERE P.Precio > 15
    AND (SELECT COUNT(*) FROM ELABORACION E WHERE E.NPizza = P.NPizza) < 4;
    ```

    La subconsulta `(SELECT COUNT(*) FROM ELABORACION E WHERE E.NPizza = P.NPizza)` es una subconsulta correlacionada: por cada pizza `P`, cuenta cuántas filas tiene en `ELABORACIÓN` (es decir, cuántos ingredientes lleva).

### 2) Pizza de más peso

Obtener el nombre de la pizza de más peso. (Si varias tienen el peso máximo, se obtendrán todas ellas).

??? success "Solución"
    El "peso" de una pizza es la suma de las cantidades de todos sus ingredientes, así que primero hay que agrupar por pizza:

    ```sql
    SELECT NPizza
    FROM ELABORACION
    GROUP BY NPizza
    HAVING SUM(Cantidad) =
        (SELECT MAX(peso)
         FROM (SELECT SUM(Cantidad) AS peso FROM ELABORACION GROUP BY NPizza) Pesos);
    ```

    Aquí no sirve una subconsulta correlacionada simple, porque lo que hay que comparar no es una columna, sino el resultado de una agregación (`SUM`) agrupada por pizza. La solución es una **subconsulta en la cláusula `FROM`** (una tabla derivada, aquí llamada `Pesos`): primero se calcula el peso de cada pizza, y sobre ese resultado se calcula el máximo.

### 3) Ingrediente con más cantidad de cada pizza

Obtener para cada pizza el ingrediente del cual lleva más cantidad.

??? success "Solución"
    ```sql
    SELECT E.NPizza, E.Ingrediente
    FROM ELABORACION E
    WHERE E.Cantidad = (SELECT MAX(E2.Cantidad) FROM ELABORACION E2 WHERE E2.NPizza = E.NPizza);
    ```

    Mismo patrón de subconsulta correlacionada que en otros ejemplos del tema de manipulación: para cada fila de `ELABORACIÓN`, compara su cantidad con la cantidad máxima de esa misma pizza.

---

## Examen 2005 (Apartado C)

De la base de datos del sistema de gestión hidrográfica nacional se extraen las siguientes relaciones:

- **RÍOS** (<u>Río</u>, CuencaHid): para cada río almacena la cuenca hidrográfica a la que pertenece administrativamente.
- **RÍOS_COM** (<u>Río, Comunidad</u>, Longitud): para cada río guarda las comunidades autónomas por las que discurre, almacenando la longitud (km) de cada río a su paso por cada comunidad.

### 1) Ríos que pasan por dos comunidades concretas

Nombres de los ríos que pasan por la "Comunidad Valenciana" y la "Comunidad de Castilla-La Mancha" (los ríos deben pasar por las dos).

??? success "Solución"
    ```sql
    SELECT DISTINCT R1.Río
    FROM RÍOS_COM R1
    WHERE R1.Comunidad = 'Comunidad Valenciana'
    AND EXISTS (SELECT * FROM RÍOS_COM R2
                WHERE R2.Río = R1.Río AND R2.Comunidad = 'Comunidad de Castilla-La Mancha');
    ```

    También se puede resolver con `INTERSECT` (disponible en MySQL 8.0.31+ / MariaDB 10.3+), que expresa aún más directamente la idea de "está en ambos conjuntos":

    ```sql
    SELECT Río FROM RÍOS_COM WHERE Comunidad = 'Comunidad Valenciana'
    INTERSECT
    SELECT Río FROM RÍOS_COM WHERE Comunidad = 'Comunidad de Castilla-La Mancha';
    ```

### 2) Río que más kilómetros discurre por la Comunidad Valenciana

Nombre del río que más kilómetros discurre por la "Comunidad Valenciana".

??? success "Solución"
    ```sql
    SELECT Río
    FROM RÍOS_COM
    WHERE Comunidad = 'Comunidad Valenciana'
    AND Longitud = (SELECT MAX(Longitud) FROM RÍOS_COM WHERE Comunidad = 'Comunidad Valenciana');
    ```

### 3) Ríos de la cuenca Pirenaica que pasan por más de una comunidad

Nombre y longitud total de los ríos pertenecientes a la cuenca hidrográfica "Pirenaica" que pasan por más de una comunidad.

??? success "Solución"
    ```sql
    SELECT R.Río, SUM(RC.Longitud) AS LongitudTotal
    FROM RÍOS R JOIN RÍOS_COM RC ON R.Río = RC.Río
    WHERE R.CuencaHid = 'Pirenaica'
    GROUP BY R.Río
    HAVING COUNT(*) > 1;
    ```

    El `WHERE` filtra primero por cuenca; el `GROUP BY` agrupa las filas de cada río para poder sumar su longitud total con `SUM`; y el `HAVING COUNT(*) > 1` se queda solo con los ríos que tienen más de una fila en `RÍOS_COM`, es decir, que pasan por más de una comunidad.

---

## Examen 2009

A partir de la base de datos de un laboratorio farmacéutico se extraen las siguientes tablas:

- **FORMULA** (<u>Nom_Formula</u>, caducidad, importe): nombre de la fórmula, periodo máximo de caducidad y el importe del producto en euros.
- **COMPOSICION** (<u>Nom_Formula, componente</u>, cantidad): almacena para cada fórmula magistral los nombres de sus componentes y la cantidad en mililitros de cada uno de ellos necesaria para la creación.

### a) Fórmulas con caducidad inferior a 30 días y más de 3 componentes

Mostrar los nombres de las fórmulas cuya caducidad sea inferior a 30 días y para su generación sean necesarios más de 3 componentes.

??? success "Solución"
    ```sql
    SELECT F.Nom_Formula
    FROM FORMULA F
    WHERE F.caducidad < 30
    AND (SELECT COUNT(*) FROM COMPOSICION C WHERE C.Nom_Formula = F.Nom_Formula) > 3;
    ```

### b) Componente de mayor cantidad de cada fórmula

Mostrar para cada una de las fórmulas el componente cuya cantidad sea la mayor de todos los componentes de dicha fórmula.

??? success "Solución"
    ```sql
    SELECT C.Nom_Formula, C.componente
    FROM COMPOSICION C
    WHERE C.cantidad = (SELECT MAX(C2.cantidad) FROM COMPOSICION C2 WHERE C2.Nom_Formula = C.Nom_Formula);
    ```

### c) Fórmula con menor presentación

Obtener la fórmula que menor presentación tenga. Existe la posibilidad de que existan varias de ellas con la misma presentación. (Se entiende por presentación la cantidad total en mililitros de la composición).

??? success "Solución"
    Igual que en el ejercicio del "peso" de la pizza (examen de 2004), la presentación es una suma agrupada, así que hace falta una tabla derivada:

    ```sql
    SELECT Nom_Formula
    FROM COMPOSICION
    GROUP BY Nom_Formula
    HAVING SUM(cantidad) =
        (SELECT MIN(presentacion)
         FROM (SELECT SUM(cantidad) AS presentacion FROM COMPOSICION GROUP BY Nom_Formula) Presentaciones);
    ```

### d) Renombrar el componente C3 a CV3

Actualizar el componente C3: debemos cambiar su nombre por CV3. Dichos cambios deben registrarse en la tabla COMPOSICIÓN.

??? success "Solución"
    ```sql
    UPDATE COMPOSICION SET componente = 'CV3' WHERE componente = 'C3';
    ```

    Es una instrucción `UPDATE` simple (DML), no una operación de `ALTER TABLE`: no se está renombrando una columna de la tabla, sino cambiando el valor almacenado en las filas donde `componente = 'C3'`.
