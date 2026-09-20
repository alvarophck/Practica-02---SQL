# Respuestas — Práctica 02: Northwind SQL

## Pregunta 1 — Catálogo comercial activo

**Enunciado:** Productos no descatalogados con precio entre 10 y 50 €, ordenados de mayor a menor precio.

**Consulta:**

```sql
-- EJ 1
SELECT product_name           AS producto,
       ROUND(unit_price::numeric, 2) AS precio
FROM products
WHERE discontinued = 0
  AND unit_price BETWEEN 10 AND 50
ORDER BY precio DESC;
```

**Resultado:**

![Resultado pregunta 1](img/p01.png)

**Comentario:** Al principio pensé que `discontinued` sería un booleano y probé con `NOT discontinued`, pero da error porque en realidad es un `integer`. Comprobé el tipo de columna en pgAdmin (panel de columnas) y ahí vi que 0 es activo y 1 descatalogado. Usé BETWEEN porque es más corto que poner dos condiciones con AND.

---

## Pregunta 2 — Concentración geográfica de la cartera

**Enunciado:** Países con 5 o más clientes, con número de ciudades distintas.

**Consulta:**

```sql
-- EJ 2
SELECT country               AS pais,
       COUNT(*)              AS num_clientes,
       COUNT(DISTINCT city)  AS num_ciudades
FROM customers
GROUP BY country
HAVING COUNT(*) >= 5
ORDER BY num_clientes DESC;
```

**Resultado:**

![Resultado pregunta 2](img/p02.png)

**Comentario:** Aquí no se puede usar WHERE porque la condición depende de un COUNT, y WHERE se aplica antes de agrupar, cuando ese conteo todavía no existe. Con HAVING el filtro se aplica después del GROUP BY, que es cuando ya tenemos el número de clientes por país.

---

## Pregunta 3 — Alerta de reposición

**Enunciado:** Productos activos en riesgo de rotura de stock.

**Consulta:**

```sql
-- EJ 3
SELECT product_name    AS producto,
       units_in_stock  AS stock,
       reorder_level   AS nivel_reposicion,
       units_on_order  AS pedido_a_proveedor,
       CASE WHEN units_in_stock = 0 THEN 'CRÍTICO' ELSE 'AVISO' END AS situacion
FROM products
WHERE discontinued = 0
  AND units_in_stock IS NOT NULL
  AND reorder_level IS NOT NULL
  AND units_in_stock <= reorder_level
ORDER BY stock;
```

**Resultado:**

![Resultado pregunta 3](img/p03.png)

**Comentario:** Si no pongo el IS NOT NULL en units_in_stock y reorder_level, algunos productos desaparecen del resultado sin dar error, porque una comparación con NULL no es ni verdadera ni falsa. Me di cuenta al comparar el número de filas con y sin ese filtro, salían menos de las que esperaba.

---

## Pregunta 4 — Ficha completa de producto

**Enunciado:** Productos de proveedores de Italia, Francia o España, con su categoría y datos del proveedor.

**Consulta:**

```sql
-- EJ 4
SELECT p.product_name  AS producto,
       cat.category_name AS categoria,
       s.company_name  AS proveedor,
       s.country       AS pais,
       s.city          AS ciudad
FROM products p
INNER JOIN categories cat ON cat.category_id = p.category_id
INNER JOIN suppliers  s   ON s.supplier_id   = p.supplier_id
WHERE s.country IN ('Italy', 'France', 'Spain')
ORDER BY s.country, p.product_name;
```

**Resultado:**

![Resultado pregunta 4](img/p04.png)

**Comentario:** Tuve que mirar el diagrama ER porque al principio intenté unir products directamente con el país y no hay ninguna relación ahí, el país está en suppliers. Al final quedan 3 tablas unidas con INNER JOIN porque solo quiero productos que sí tengan categoría y proveedor.

---

## Pregunta 5 — Detalle valorizado de un pedido

**Enunciado:** Detalle línea a línea del pedido 10248.

**Consulta:**

```sql
-- EJ 5
SELECT c.company_name   AS cliente,
       o.order_date     AS fecha_pedido,
       p.product_name   AS producto,
       od.unit_price    AS precio_unitario,
       od.quantity      AS cantidad,
       od.discount      AS descuento,
       ROUND((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric), 2) AS importe_linea
FROM orders o
INNER JOIN customers c USING (customer_id)
INNER JOIN order_details od USING (order_id)
INNER JOIN products p USING (product_id)
WHERE o.order_id = 10248;
```

**Resultado:**

![Resultado pregunta 5](img/p05.png)

**Comentario:** Usé USING porque order_id y product_id se llaman igual en las dos tablas que uno cada vez, así no tengo que escribir la condición completa con ON y encima no me sale la columna duplicada en el resultado.

---

## Pregunta 6 — Ranking de categorías por facturación

**Enunciado:** Facturación total por categoría, solo las que superan 100.000 €.

**Consulta:**

```sql
-- EJ 6
SELECT cat.category_name                     AS categoria,
       COUNT(od.order_id)                    AS num_lineas,
       COUNT(DISTINCT p.product_id)          AS num_productos,
       ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)), 2) AS facturacion
FROM categories cat
INNER JOIN products p      ON p.category_id = cat.category_id
INNER JOIN order_details od ON od.product_id = p.product_id
GROUP BY cat.category_name
HAVING SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)) > 100000
ORDER BY facturacion DESC;
```

**Resultado:**

![Resultado pregunta 6](img/p06.png)

**Comentario:** Aquí repetí la misma expresión de SUM en el HAVING que en el SELECT, no encontré forma de usar el alias directamente en HAVING (en PostgreSQL no funciona). Me sorprendió que solo unas pocas categorías pasen del umbral de 100.000, pensaba que serían más.

---

## Pregunta 7 — Clientes sin actividad comercial

**Enunciado:** Todos los clientes con su número de pedidos, incluidos los que nunca han comprado.

**Consulta:**

```sql
-- EJ 7
SELECT c.company_name AS cliente,
       c.country      AS pais,
       COUNT(o.order_id) AS num_pedidos,
       COALESCE(MAX(o.order_date)::text, 'SIN PEDIDOS') AS ultimo_pedido
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.company_name, c.country
ORDER BY num_pedidos, cliente;
```

**Resultado:**

![Resultado pregunta 7](img/p07.png)

**Comentario:** Si pongo COUNT(*) en vez de COUNT(o.order_id), los clientes sin pedidos salen con un 1 en vez de un 0, porque el LEFT JOIN sí genera una fila para ellos aunque esté rellena de NULL. COUNT(columna) en cambio no cuenta los NULL, así que es la forma correcta de contar pedidos reales.

---

## Pregunta 8 — Organigrama de la fuerza de ventas

**Enunciado:** Cada empleado con su responsable directo (auto-JOIN).

**Consulta:**

```sql
-- EJ 8
SELECT emp.first_name || ' ' || emp.last_name AS empleado,
       emp.title AS cargo,
       COALESCE(jefe.first_name || ' ' || jefe.last_name, 'DIRECCIÓN GENERAL') AS responsable,
       COALESCE(jefe.title, '') AS cargo_responsable
FROM employees emp
LEFT JOIN employees jefe ON jefe.employee_id = emp.reports_to
ORDER BY empleado;
```

**Resultado:**

![Resultado pregunta 8](img/p08.png)

**Comentario:** La parte que más me costó fue entender que employees se une consigo misma, así que hacen falta dos alias distintos (emp y jefe) aunque sea la misma tabla. Usé LEFT JOIN y no INNER porque si no, el empleado que no reporta a nadie desaparecería del resultado.

---

## Pregunta 9 — Rejilla de cobertura categoría × año

**Enunciado:** Facturación por categoría y año, sin huecos (24 filas).

**Consulta:**

```sql
-- EJ 9
WITH categorias AS (
    SELECT category_id, category_name FROM categories
),
anios AS (
    SELECT generate_series(1996, 1998) AS anio
),
grid AS (
    SELECT c.category_id, c.category_name, a.anio
    FROM categorias c
    CROSS JOIN anios a
),
ventas AS (
    SELECT p.category_id,
           EXTRACT(YEAR FROM o.order_date)::int AS anio,
           SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)) AS facturacion
    FROM order_details od
    INNER JOIN orders o    ON o.order_id = od.order_id
    INNER JOIN products p  ON p.product_id = od.product_id
    GROUP BY p.category_id, EXTRACT(YEAR FROM o.order_date)
)
SELECT g.category_name AS categoria,
       g.anio,
       ROUND(COALESCE(v.facturacion, 0), 2) AS facturacion
FROM grid g
LEFT JOIN ventas v ON v.category_id = g.category_id AND v.anio = g.anio
ORDER BY categoria, anio;
```

**Resultado:**

![Resultado pregunta 9](img/p09.png)

**Comentario:** Primero genero todas las combinaciones posibles de categoría y año con CROSS JOIN (esto me da las 24 filas seguro), y luego le pego los datos reales con LEFT JOIN. Si lo hiciera al revés, agrupando directamente las ventas, las combinaciones sin ventas ese año no aparecerían nunca en el resultado.

---

## Pregunta 10 — Mapa de países: clientes frente a proveedores

**Enunciado:** Clientes y proveedores por país, incluyendo los que solo tienen uno u otro.

**Consulta:**

```sql
-- EJ 10
WITH clientes_pais AS (
    SELECT country AS pais, COUNT(*) AS num_clientes
    FROM customers
    GROUP BY country
),
proveedores_pais AS (
    SELECT country AS pais, COUNT(*) AS num_proveedores
    FROM suppliers
    GROUP BY country
)
SELECT COALESCE(cp.pais, pp.pais) AS pais,
       COALESCE(cp.num_clientes, 0)    AS num_clientes,
       COALESCE(pp.num_proveedores, 0) AS num_proveedores,
       CASE
           WHEN cp.pais IS NOT NULL AND pp.pais IS NOT NULL THEN 'AMBOS'
           WHEN cp.pais IS NOT NULL THEN 'SOLO CLIENTES'
           ELSE 'SOLO PROVEEDORES'
       END AS tipo_presencia
FROM clientes_pais cp
FULL JOIN proveedores_pais pp ON pp.pais = cp.pais
ORDER BY pais;
```

**Resultado:**

![Resultado pregunta 10](img/p10.png)

**Comentario:** Si en el SELECT pongo solo cp.pais, los países que solo existen en proveedores saldrían en NULL, porque en un FULL JOIN esa columna puede venir vacía de cualquiera de los dos lados. Con COALESCE cojo el que no sea nulo de los dos.

---

## Pregunta 11 — Directorio unificado de contactos

**Enunciado:** Contactos de clientes, proveedores y empleados en una sola tabla.

**Consulta:**

```sql
-- EJ 11
SELECT 'CLIENTE'   AS origen, UPPER(contact_name) AS contacto, company_name AS organizacion, city AS ciudad, country AS pais
FROM customers
UNION ALL
SELECT 'PROVEEDOR', UPPER(contact_name), company_name, city, country
FROM suppliers
UNION ALL
SELECT 'EMPLEADO', UPPER(first_name || ' ' || last_name), 'NORTHWIND TRADERS', city, country
FROM employees
ORDER BY origen, pais;
```

**Resultado:**

![Resultado pregunta 11](img/p11.png)

**Comentario:** Usé UNION ALL porque UNION quita filas duplicadas, y aquí no tiene sentido: si un cliente y un proveedor casualmente tuvieran el mismo nombre de contacto y ciudad, UNION los fusionaría en una fila y perderíamos uno de los dos contactos sin darnos cuenta.

---

## Pregunta 12 — Mercados con desequilibrio

**Enunciado:** a) Países con clientes pero sin proveedores. b) Países con ambos.

**Consulta (a):**

```sql
-- EJ 12a
SELECT country AS pais FROM customers
EXCEPT
SELECT country FROM suppliers
ORDER BY pais;
```

**Resultado (a):**

![Resultado pregunta 12a](img/p12a.png)

**Consulta (b):**

```sql
-- EJ 12b
SELECT country AS pais FROM customers
INTERSECT
SELECT country FROM suppliers
ORDER BY pais;
```

**Resultado (b):**

![Resultado pregunta 12b](img/p12b.png)

**Comentario:** Con EXCEPT e INTERSECT el código queda mucho más corto que con un LEFT JOIN ... WHERE ... IS NULL, y además estos operadores quitan duplicados automáticamente. Probé también la versión con LEFT JOIN y el resultado es el mismo, solo que hay que acordarse de filtrar por la columna de la derecha siendo NULL para quedarte con los que no tienen proveedor.
