# PostgreSQL Advanced Queries - Rollup and Cube Operations


For starter lets see a simple example of grouping data by country and category

```sql
SELECT
    o.ship_country,
    c.category_name,
    SUM(od.price)::numeric(10,2)
FROM
    od_w_p od
JOIN
    orders o ON od.order_id = o.order_id
JOIN
    products p ON od.product_id = p.product_id
JOIN
    categories c ON p.category_id = c.category_id
GROUP BY
    o.ship_country,
    c.category_id,
    c.category_name
ORDER BY
    o.ship_country,
    c.category_name;
```    

```sql
SELECT
    o.ship_country,
    null category_name,
    SUM(od.price)::numeric(10,2)
FROM
    od_w_p od
JOIN
    orders o ON od.order_id = o.order_id
JOIN
    products p ON od.product_id = p.product_id
JOIN
    categories c ON p.category_id = c.category_id
GROUP BY
    o.ship_country
ORDER BY
    o.ship_country
```


	

```sql
-- Use ROLLUP to get subtotals for each country and category
SELECT
    o.ship_country,
    c.category_name,
    SUM(od.price)::numeric(10,2)
FROM
    od_w_p od
JOIN
    orders o ON od.order_id = o.order_id
JOIN
    products p ON od.product_id = p.product_id
JOIN
    categories c ON p.category_id = c.category_id
GROUP BY ROLLUP (
    o.ship_country,
    c.category_name
)
ORDER BY
    o.ship_country NULLS LAST, -- NULLS LAST places the rollup totals at the end of each country group
    c.category_name NULLS LAST; -- NULLS LAST places the grand total at the end	
```

```sql
-- Use CUBE to get subtotals for each country and category
SELECT
    o.ship_country,
    c.category_name,
    SUM(od.price)::numeric(10,2)
FROM
    od_w_p od
JOIN
    orders o ON od.order_id = o.order_id
JOIN
    products p ON od.product_id = p.product_id
JOIN
    categories c ON p.category_id = c.category_id
GROUP BY CUBE (
    o.ship_country,
    c.category_name
)
ORDER BY
    o.ship_country NULLS LAST, -- NULLS LAST places the rollup totals at the end of each country group
    c.category_name NULLS LAST; -- NULLS LAST places the grand total at the end	
	


	