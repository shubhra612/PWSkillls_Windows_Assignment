use mavenmovies;
-- 1. Rank the customers based on the total amount they've spent on rentals.

select concat_ws(' ',c.first_name,c.last_name) as customers, sum(p.amount) as amount, 
rank() over (order by sum(p.amount)) as Ranks
 from customer c 
left join payment p
on c.customer_id=p.customer_id
group by concat_ws(' ',c.first_name,c.last_name);

-- 2. Calculate the cumulative revenue generated by each film over time.
SELECT film_id, title, rental_date, amount,
       SUM(amount) OVER (PARTITION BY film_id ORDER BY rental_date) AS cumulative_revenue
FROM (
    SELECT f.film_id, f.title, r.rental_date, p.amount
    FROM film f
    JOIN inventory i ON f.film_id = i.film_id
    JOIN rental r ON i.inventory_id = r.inventory_id
    JOIN payment p ON r.rental_id = p.rental_id
) AS film_revenue;

-- 3. Determine the average rental duration for each film, considering films with similar lengths.
SELECT film_id, title, rental_duration,
AVG(rental_duration) OVER (PARTITION BY rental_duration) AS avg_rental_duration
FROM film;


-- 4. Identify the top 3 films in each category based on their rental counts.

WITH category_film_rentals AS (
    SELECT fc.category_id, f.film_id, f.title, COUNT(r.rental_id) AS rental_count,
           DENSE_RANK() OVER (PARTITION BY fc.category_id ORDER BY COUNT(r.rental_id) DESC) AS rank_in_category
    FROM film f
    JOIN film_category fc ON f.film_id = fc.film_id
    JOIN inventory i ON f.film_id = i.film_id
    JOIN rental r ON i.inventory_id = r.inventory_id
    GROUP BY fc.category_id, f.film_id, f.title
)
SELECT category_id, film_id, title, rental_count, rank_in_category
FROM category_film_rentals
WHERE rank_in_category <= 3;
/* 5. Calculate the difference in rental counts between each customer's total rentals and
 the average rentals across all customers.*/
 
select distinct customer_id,
count(rental_id) over
(partition by customer_id order by customer_id) -(select count(rental_id)/count(distinct(customer_id)) from rental) as rental_diff
 from rental;
 
-- 6. Find the monthly revenue trend for the entire rental store over time.
 
 SELECT DATE_FORMAT(payment_date, '%Y-%m') AS payment_month,
       SUM(amount) AS monthly_revenue
FROM payment
GROUP BY payment_month
ORDER BY payment_month; 
 
 -- 7. Identify the customers whose total spending on rentals falls within the top 20% of all customers.
 
 with cte as 
(select c.customer_id, ntile(5) over (order by p.amount) as which_tile from customer c
left join inventory i on c.store_id= i.store_id
left join rental r on r.inventory_id= i.inventory_id
left join payment p on r.rental_id= p.rental_id)
select customer_id from cte where which_tile= 5 ;

-- 8. Calculate the running total of rentals per category, ordered by rental count.

select count(r.rental_id) over(partition by c.category_id order by r.rental_id) as running_total_rentals
from rental r
left join inventory i on r.inventory_id= i.inventory_id
left join film_category c on i.film_id= c.film_id;

-- 9. Find the films that have been rented less than the average rental count for their respective categories.

WITH category_avg_rentals AS (
    SELECT fc.category_id, 
           AVG(COUNT(r.rental_id)) OVER (PARTITION BY fc.category_id) AS avg_rental_count
    FROM film f
    JOIN film_category fc ON f.film_id = fc.film_id
    JOIN inventory i ON f.film_id = i.film_id
    JOIN rental r ON i.inventory_id = r.inventory_id
    GROUP BY fc.category_id, f.film_id, f.title
)
SELECT cf.category_id, cf.film_id, cf.title, cf.rental_count, car.avg_rental_count
FROM (
    SELECT fc.category_id, f.film_id, f.title, COUNT(r.rental_id) AS rental_count
    FROM film f
    JOIN film_category fc ON f.film_id = fc.film_id
    JOIN inventory i ON f.film_id = i.film_id
    JOIN rental r ON i.inventory_id = r.inventory_id
    GROUP BY fc.category_id, f.film_id, f.title
) AS cf
JOIN category_avg_rentals AS car ON cf.category_id = car.category_id
WHERE cf.rental_count < car.avg_rental_count; 

-- 10. **Identify the top 5 months with the highest revenue and display the revenue generated in each month:**


SELECT payment_month, monthly_revenue
FROM (
    SELECT DATE_FORMAT(payment_date, '%Y-%m') AS payment_month,
           SUM(amount) AS monthly_revenue,
           RANK() OVER (ORDER BY SUM(amount) DESC) AS revenue_rank
    FROM payment
    GROUP BY DATE_FORMAT(payment_date, '%Y-%m')
) AS monthly_revenues
WHERE revenue_rank <= 5;
