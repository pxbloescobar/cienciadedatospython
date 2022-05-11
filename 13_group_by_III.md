# Agrupaciones y agregaciones III

Es vaaaasto el tema, eh?

## Filtrando grupos con `having`

Como hemos visto empíricamente, las reglas del `having` son las siguientes:

1. Mientras que `where` **filtra rows**, el `having` **filtra grupos**.
2. El `having` se ejecuta después del `group by` y antes del `select`.
3. Al usar el `group by`, frecuentemente usaremos una función de agregación en el `select` como `avg`, `sum`, etc, y frecuentemente le asignaremos un alias a esa función.
4. Pero **no podemos usar este alias** en el having, sino que **tenemos que repetir la función** y además agregarle una condición para que sea una evaluación booleana.
5. La sintaxis de estas condiciones están gobernadas por los mismos operadores de igualdad (`<`, `>`, `=`, etc) que como si los usáramos en el `where`.

Pues resulta que no tan vaaaaaasto, no?

## Ejercicios con la BD Sakila

### 1. Cuantos actores comparten apellido con al menos otro actor?
Si queremos solo listar los apellidos repetidos (apellidos que pertenecen a >=2 actores):
```
select a.last_name, count(a.actor_id) actores_apellido_repetido 
from actor a
group by a.last_name 
having count(a.actor_id) > 1
```

Si queremos CONTAR los apellidos repetidos:
```
select count(t.last_name) from (
	select a.last_name, count(a.actor_id) actores_apellido_repetido  from actor a
	group by a.last_name 
	having count(a.actor_id) > 1
	) as t;
```

Si queremos contar LOS ACTORES con apellidos repetidos:
```
select sum(actores_apellido_repetido) from (
	select a.last_name, count(a.actor_id) actores_apellido_repetido  from actor a
	group by a.last_name 
	having count(a.actor_id) > 1
	) as t;
```
### 2. De nuestros empleados, cuál es el que más negocio trajo a nuestro store en 2005?
```
select s.first_name, s.last_name, sum(p.amount)
from staff s join payment p using (staff_id)
WHERE extract(year from p.payment_date) = 2005
group by s.first_name, s.last_name;
```

### 3. Cuantos películas tienen un ensemble cast (5 o más actores 5⭐)?
Si queremos listar las películas con 5 o más actores:
```
select f.title, f.film_id , count(*) actores
from film f join film_actor fa using (film_id)
group by f.film_id 
having count(*) >= 5
order by actores desc
```

Si queremos contar las películas con 5 o más actores:
```
select count(t.title) from (
	select f.title, f.film_id , count(*) actores
	from film f join film_actor fa using (film_id)
	group by f.film_id 
	having count(*) >= 5
	order by actores desc
	) as t
```
### 4. Cuál ha sido nuestro cliente más redituable cada año?
Solución con subselect:
```
select distinct on (t.year_payment) t.year_payment, t.sum_amount as max_sum_amount from (
	select p.customer_id, sum(p.amount) sum_amount, extract(year from p.payment_date) as year_payment
	from payment p join customer c using (customer_id)
	group by p.customer_id, year_payment
	) as t
	order by t.year_payment, max_sum_amount desc
```

Solución con puro `distinct on` contribuído por @jorcazas
```
select distinct on (payment_year) extract(year from p.payment_date) as payment_year, c.customer_id, sum(p.amount) as sum_amount 
from customer c join payment p using (customer_id) 
group by c.customer_id, payment_year
order by payment_year, sum_amount desc;
```

Por qué SÍ✔️ funciona esto con `distinct on`? Siguiendo la secuencia de ejecución:

1. **`from`**: dataset conjuntando `customer` y `payment`. Tendremos `customer` repetidos por la relación de 1 `customer` VS N `payment`.
2. **`group by`**: armamos grupos de `customer_id` y `payment_year`. Cada par es único. `payment_year` se obtiene con `extract(year from payment_date)`. Dado que tenemos N `payment`, y por tanto, N `payment_year` para 1 `customer`, entonces los grupos tendrán el `customer` repetido y todos los años por separado.
3. **`select`**: seleccionamos el `year` extraído de `payment_date`, el `customer_id`, la suma de los montos de los pagos **por cada grupo dado por `group by`** calculado con `sum(amount)`.
4. **`distinct on`**: desduplicamos `payment_year` y apuntamos al 1er registro que corresponda a cada año desduplicado.
5. **`order by`**: ordenamos **por grupo** de forma descendente, los años y la suma de los pagos dado por `sum_amount`. **ESTO** es lo que garantiza que el 1er registro obtenido **DESPUÉS** de la desduplicación con `distinct on` sea el máximo de fecha y el máximo de pago, pero esto no es global, sino es **por grupo** por la presencia del `group by`.

### 5. Cuales películas cuyo título comiencen con la letra Q o K están en idioma inglés?
La respuesta es esta:
```
select f.title, l."name"  
from film f join "language" l using (language_id)
where l."name" = 'English' and (f.title like 'K%' or f.title like 'Q%');
```

Pero un query que nos da los mismos resultados, pero que **está incorrecta**, es la siguiente:
```
select f.title , l."name" 
from film f join "language" l using (language_id)
where l."name" = 'English'
group by f.title , l."name" 
having f.title like 'Q%' or f.title like 'K%'
```

Por qué esto funciona aunque esté mal⚠️? Sigamos la secuencia de ejecución:

1. **`from`**: obtenemos el dataset uniendo `film` y `language`, el cual tendrá `film.title` repetidos porque 1 `film` puede tener N `language`.
2. **`where`**: filtramos rows y nos quedamos solo los que tengan `language = 'English'`, lo cual implica que **desduplicamos** las películas y nos quedamos con películas únicas, todas en inglés
3. **`group by`**: hacemos grupos tomando el título de la película y el lenguaje, pero dado (2), **cada grupo solo tiene 1 renglón**, porque ya tenemos películas únicas. Esto es incorrecto. Cuando los grupos solo tienen 1 observación, es que están mal armados o que hay que mover esa lógica a otras cláusulas, como `select`.
4. **`having`**: recordemos que `having` filtra grupos, pero como nuestros grupos son **renglones individuales**, entonces actúa como si fuera un `where`.
5. Y por eso el resultado es idéntico.

## Tarea

Usando la BD de Sakila, y en un script de SQL separado, y **en su propio repo de Github**, escribir los queries necesarios y suficientes para dar respuesta a las siguientes preguntas:

1. Cómo obtenemos todos los nombres y correos de nuestros clientes canadienses para una campaña?
2. Qué cliente ha rentado más de nuestra sección de adultos?
3. Qué películas son las más rentadas en todas nuestras stores?
4. Cuál es nuestro revenue por store?

**Timestamp límite de entrega:** Lunes 16 de Mayo, a las 12:59:59 del medio día.

