# Sakila Úkol

## 1) Filmy, které se půjčily za poslední rok

```sql
SELECT DISTINCT 
	customer.first_name, customer.last_name, address.address
FROM 
	rental
JOIN 
	inventory ON rental.inventory_id = inventory.inventory_id
JOIN 
	film ON inventory.film_id = film.film_id
JOIN 
	customer ON rental.customer_id = customer.customer_id
JOIN 
	address ON customer.address_id = address.address_id
WHERE 
	rental.rental_date >= NOW() - INTERVAL '1 year'
```
## 2) Kolik filmů je v jaké kategorii
```sql
SELECT 
	category.name, COUNT(film_category.film_id) AS film_count
FROM 
	category
LEFT JOIN 
	film_category ON category.category_id = film_category.category_id
GROUP BY 
	category.name
ORDER BY
	film_count DESC;
```
## 3) Pokuta
### Pomocná funkce
```sql
CREATE OR REPLACE FUNCTION calculate_late_fee(
    p_rental_date TIMESTAMP,
    p_return_date TIMESTAMP,
    p_rental_rate DECIMAL,
    p_days_allowed INTEGER
) RETURNS DECIMAL AS $$
DECLARE
    days_late INTEGER;
BEGIN
    -- If return_date is NULL, consider NOW() as the return date
    IF p_return_date IS NULL THEN
        p_return_date := NOW();
    END IF;

    days_late := GREATEST(DATE_PART('day', p_return_date - p_rental_date) - p_days_allowed, 0);
    RETURN p_rental_rate * 0.01 * days_late;
END;
$$ LANGUAGE PLPGSQL;
```
### Samotné Query

```sql
SELECT 
	calculate_late_fee(rental.rental_date, rental.return_date, film.rental_rate, 14) AS late_fee,
	customer.last_name,
	customer.first_name,
	film.title,
	rental.rental_date,
	rental.return_date
FROM
	rental
JOIN
    inventory ON rental.inventory_id = inventory.inventory_id
JOIN
    film ON inventory.film_id = film.film_id
JOIN
	customer ON rental.customer_id = customer.customer_id
ORDER BY 
	late_fee DESC
```
### Pozorování...
- vypadá to, že kromě těch, kteří film nikdy nevrátili, nikdo žádnou pokutu nedostal
- zkoušel jsem snížit laťku na 9 dní, tam už šly vidět i další výsledky

## 4) Kolik kdo prodal v každém roce

```sql
SELECT
    EXTRACT(YEAR FROM rental.rental_date) AS rental_year,
    staff.staff_id,
    staff.first_name || ' ' || staff.last_name AS staff_name,
    COUNT(*) AS rental_count
FROM
    rental
JOIN
    staff ON rental.staff_id = staff.staff_id
GROUP BY
    EXTRACT(YEAR FROM rental.rental_date),
    staff.staff_id,
    staff.first_name,
    staff.last_name
ORDER BY
    EXTRACT(YEAR FROM rental.rental_date) DESC,
    staff.staff_id;

```

## 5) Funkce pro přidání filmu
### Definice funkce
```sql
CREATE OR REPLACE FUNCTION add_new_film(
    p_title VARCHAR(255),
    p_description TEXT,
    p_release_year YEAR,
    p_language_id INT,
    p_original_language_id INT,
    p_rental_duration SMALLINT,
    p_rental_rate NUMERIC(5,2),
    p_length SMALLINT,
    p_replacement_cost NUMERIC(6,2),
    p_rating mpaa_rating,
    p_special_features TEXT[],
    p_fulltext TSVECTOR
)
RETURNS VOID AS $$
BEGIN
    -- Vložení nového filmu do tabulky
    INSERT INTO film (
        title,
        description,
        release_year,
        language_id,
        original_language_id,
        rental_duration,
        rental_rate,
        length,
        replacement_cost,
        rating,
        special_features,
        last_update,
        fulltext
    ) VALUES (
        p_title,
        p_description,
        p_release_year,
        p_language_id,
        p_original_language_id,
        p_rental_duration,
        p_rental_rate,
        p_length,
        p_replacement_cost,
        p_rating,
        p_special_features,
        CURRENT_TIMESTAMP,
        p_fulltext
    );
END;
$$ LANGUAGE PLPGSQL;

```
### Příklad použití
```sql
SELECT add_new_film(
    p_title => 'The Title',
    p_description => 'Description of the film',
    p_release_year => 2022::YEAR,
    p_language_id => 1::INT,
    p_original_language_id => 2::INT,
    p_rental_duration => 7::SMALLINT,
    p_rental_rate => 4.99::NUMERIC(5,2),
    p_length => 120::SMALLINT,
    p_replacement_cost => 19.99::NUMERIC(6,2),
    p_rating => 'PG'::mpaa_rating,
    p_special_features => NULL::text[],
    p_fulltext => to_tsvector('Full text of the film')
);
```
### Ověření vložení
```sql
SELECT 
	*
FROM
	film
WHERE
	film.title = 'The Title'
```
