# Pritam_Shetty

-- using database
USE `gdb023`;


-- Request 1. Provide the list of markets in which customer "Atliq Exclusive" operates its business in the APAC region.


SELECT DISTINCT(market) AS Atliq_Exclusive_Operating_Markets
    
FROM


    dim_customer
    
    
WHERE


    customer = 'Atliq Exclusive'
        AND region = 'apac'
        
        
ORDER BY Atliq_Exclusive_Operating_Markets;


-- Request 2. What is the percentage of unique product increase in 2021 vs. 2020?


-- finding the total number of products in 2020



WITH unique_2020 AS


(


SELECT 
    COUNT(DISTINCT (product_code)) AS unique_products_2020
    
    
FROM
    fact_sales_monthly
    
    
WHERE
    fiscal_year = 2020)
    
    
    
    -- combining the 2020 product count with 2021 and finding % increase
    
SELECT 


    unique_products_2020,
    COUNT(DISTINCT (product_code)) AS unique_products_2021,
    ROUND((((COUNT(DISTINCT (product_code)) - unique_products_2020) / unique_products_2020) * 100),
            2) AS percentage_chg
            
            
FROM
    unique_2020,
    fact_sales_monthly
    
    
WHERE
    fiscal_year = 2021;
    
    

-- Request 3. Provide a report with all the unique product counts for each segment and sort them in descending order of product counts.

SELECT segment,
       Count(DISTINCT ( product_code )) AS product_count
FROM   dim_product
GROUP  BY segment
ORDER  BY product_count DESC; 



-- Request 4. Follow-up: Which segment had the most increase in unique products in 2021 vs 2020?

-- calculating 2020 products

WITH product_2020
     AS (SELECT segment,
                Count(DISTINCT ( product_code )) AS product_count_2020
         FROM   dim_product
                INNER JOIN fact_sales_monthly using (product_code)
         WHERE  fiscal_year = 2020
         GROUP  BY segment),
         
     -- calculating 2021 products
     
     product_2021
     AS (SELECT segment,
                Count(DISTINCT ( product_code )) AS product_count_2021
         FROM   dim_product
                LEFT JOIN fact_sales_monthly using (product_code)
         WHERE  fiscal_year = 2021
         GROUP  BY segment)
         
-- combining both the tables

SELECT segment,
       product_count_2020,
       product_count_2021,
       ( product_count_2021 - product_count_2020 ) AS difference
FROM   product_2020
       INNER JOIN product_2021 using (segment)
ORDER  BY difference DESC; 

-- Request 5. Get the products that have the highest and lowest manufacturing costs.

-- creating a table and where products are ranked according to the manufacturing cost

WITH min_max_cost
     AS (SELECT product_code,
                product,
                manufacturing_cost,
                Rank()
                  OVER (
                    ORDER BY manufacturing_cost)      min_manu_cost,
                Rank()
                  OVER (
                    ORDER BY manufacturing_cost DESC) max_manu_cost
         FROM   dim_product
                INNER JOIN fact_manufacturing_cost using(product_code))
                
-- displaying the required data 

SELECT product_code,
       product,
       round(manufacturing_cost,2) as manufacturing_cost
FROM   min_max_cost
WHERE  min_manu_cost = 1
        OR max_manu_cost = 1
ORDER  BY manufacturing_cost DESC; 

-- Request 6. Generate a report which contains the top 5 customers who received an average high pre_invoice_discount_pct for the fiscal year 2021 and in the
-- Indian market.

SELECT customer_code,
       customer,
       round(Avg(pre_invoice_discount_pct),2) AS average_discount_percentage
FROM   fact_pre_invoice_deductions
       INNER JOIN dim_customer USING(customer_code)
WHERE  fiscal_year = 2021
       AND market = 'India'
GROUP  BY customer_code
ORDER  BY average_discount_percentage DESC
LIMIT  5; 

-- Request 7. Get the complete report of the Gross sales amount for the customer “Atliq Exclusive” for each month.

-- finding monthwise Gross Sales for 2020

WITH fiscal_2020
     AS (SELECT Month(date)                                              AS
                Month,
                fsm.fiscal_year                                          AS Year
                ,
                Round(Sum(( gross_price * sold_quantity )) /
                      1000000, 2) AS
                'Gross_sales_Amount(millions)'
         FROM   dim_customer
                INNER JOIN fact_sales_monthly fsm using(customer_code)
                INNER JOIN fact_gross_price using(product_code)
         WHERE  customer = 'Atliq Exclusive'
                AND fsm.fiscal_year = 2020
         GROUP  BY month),
         
     -- finding monthwise gross sales for 2021
     
     fiscal_2021
     AS (SELECT Month(date)                                              AS
                Month,
                fsm.fiscal_year                                          AS Year
                ,
                Round(Sum(( gross_price * sold_quantity )) /
                      1000000, 2) AS
                'Gross_sales_Amount(millions)'
         FROM   dim_customer
                INNER JOIN fact_sales_monthly fsm using(customer_code)
                INNER JOIN fact_gross_price using(product_code)
         WHERE  customer = 'Atliq Exclusive'
                AND fsm.fiscal_year = 2021
         GROUP  BY month)
         
-- combining both tables

SELECT *
FROM   fiscal_2020
UNION ALL
SELECT *
FROM   fiscal_2021; 



-- Request 8. In which quarter of 2020, got the maximum total_sold_quantity?

SELECT Quarter(date)      AS Quarter,
       round(Sum(sold_quantity)/1000000,2) AS 'total_sold_quantity_million'
FROM   fact_sales_monthly
WHERE  Year(date) = 2020
GROUP  BY Quarter(date)
ORDER  BY total_sold_quantity_million DESC
; 

-- Months between Oct- December highest sold quantity

-- Request 9. Which channel helped to bring more gross sales in the fiscal year 2021 and the percentage of contribution?

-- finding the value of gross sale

WITH gross_sales AS
(
           SELECT     Sum(gross_price * sold_quantity)/1000000 AS gross_sales
           FROM       fact_gross_price
           INNER JOIN fact_sales_monthly fsm
           using     (product_code)
           WHERE      fsm.fiscal_year = 2021)
           
-- using the gross sale value and finding the channel which created highest sales.

SELECT     channel,
           Round(Sum(gross_price * sold_quantity) / 1000000, 2)                         AS gross_sales_mln,
           Round(((Sum(gross_price * sold_quantity) / 1000000) / gross_sales) * 100, 2) AS percentage
FROM       gross_sales,
           dim_customer
INNER JOIN fact_sales_monthly fsm
using      (customer_code)
INNER JOIN fact_gross_price
using      (product_code)
WHERE      fsm.fiscal_year = 2021
GROUP BY   channel
ORDER BY   gross_sales_mln DESC;

-- Request 10 Get the Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021?

-- generating told sold quantity

WITH total_sold_quantity
     AS (SELECT division,
                product_code,
                product,
                Sum(sold_quantity)                    AS total_sold_quantity,
                Rank()
                  OVER(
                    partition BY division
                    ORDER BY Sum(sold_quantity) DESC) AS rank_order
         FROM   dim_product
                INNER JOIN fact_sales_monthly using(product_code)
         GROUP  BY product_code)

-- displaying the the top 3 product under each division    

SELECT *
FROM   total_sold_quantity
HAVING rank_order <= 3; 
