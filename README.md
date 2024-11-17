use elec_vicals;

-- Data cleaning

alter table electric_vehicle_sales_by_makers
rename to ev_sales_by_makers;

ALTER TABLE ev_sales_by_makers
RENAME COLUMN ï»¿date to date;

ALTER TABLE ev_sales_by_state
RENAME COLUMN ï»¿date to date;

ALTER TABLE dim_date
RENAME COLUMN ï»¿date to date;

alter table electric_vehicle_sales_by_state
rename to ev_sales_by_state;


ALTER table ev_sales_by_makers
MODIFY COLUMN date DATE;
DESC dim_date;



select*from ev_sales_by_makers;
select*from ev_sales_by_state;
SELECT * from dim_date;


--  List the top 3 and bottom 3 makers for the fiscal years 2023 and 2024 in 
--  terms of the number of 2-wheelers sold. 


WITH top_maker AS (
    SELECT 
        sm.maker, 
        SUM(sm.electric_vehicles_sold) AS vihcle_sold,
        DENSE_RANK() OVER (ORDER BY SUM(sm.electric_vehicles_sold)DESC ) AS top_selar
    FROM 
        ev_sales_by_makers sm
    JOIN 
        dim_date d 
    ON 
        d.date = sm.date
    WHERE 
        d.fiscal_year IN (2023, 2024)
        AND sm.vehicle_category = '2-wheelers'
    GROUP BY 
        sm.maker
    ORDER BY 
         vihcle_sold DESC
)
SELECT 
    maker, 
    vihcle_sold 
FROM 
    top_maker
WHERE 
    top_selar <= 3;


-- Identify the top 5 states with the highest penetration rate in 2-wheeler 
-- and 4-wheeler EV sales in FY 2024. 



WITH top_state as(
SELECT 
    se.state,
    (SUM(se.electric_vehicles_sold) / SUM(se.total_vehicles_sold)) * 100 AS p_rate -- Penetration rate
    , DENSE_RANK() OVER(ORDER BY (SUM(se.electric_vehicles_sold) / SUM(se.total_vehicles_sold)) * 100 desc)as p_rank -- Penetration rank 
FROM 
    ev_sales_by_state se
JOIN 
    dim_date dd ON dd.date = se.date
WHERE 
    se.vehicle_category IN ('2-wheelers', '4-wheelers') 
    AND dd.fiscal_year = 2024 
GROUP BY 
    se.state
ORDER BY 
    p_rate DESC)

SELECT state , p_rate FROM top_state 
WHERE p_rank <=5;


-- List the states with negative penetration (decline) in EV sales from 2022 
--  to 2024? 

    

WITH top_state AS (
    SELECT 
        se.state,
        dd.fiscal_year,
        (SUM(se.electric_vehicles_sold) / SUM(se.total_vehicles_sold)) * 100 AS p_rate, -- Penetration rate
        DENSE_RANK() OVER(ORDER BY (SUM(se.electric_vehicles_sold) / SUM(se.total_vehicles_sold)) * 100 DESC) AS p_rank -- Penetration rank 
    FROM 
        ev_sales_by_state se
    JOIN 
        dim_date dd ON dd.ï»¿date = se.ï»¿date
    WHERE 
        se.vehicle_category IN ('2-wheelers', '4-wheelers') 
        AND dd.fiscal_year BETWEEN 2022 and 2024
    GROUP BY 
        se.state, dd.fiscal_year
),
penetration_change AS (
    SELECT 
        sp_2022.state,
        sp_2022.p_rate AS p_rate_2022,
        sp_2024.p_rate AS p_rate_2024,
        (sp_2022.p_rate - sp_2024.p_rate) AS change_in_p_rate
    FROM 
        top_state sp_2022
    JOIN 
        top_state sp_2024 ON sp_2022.state = sp_2024.state
    WHERE 
        sp_2022.fiscal_year = 2022 
        AND sp_2024.fiscal_year = 2024
)
SELECT 
    pc.state, 
    pc.change_in_p_rate
FROM 
    penetration_change pc
WHERE
	change_in_p_rate < 0

ORDER BY 
    pc.change_in_p_rate;


-- What are the quarterly trends based on sales volume for the top 5 EV 
-- makers (4-wheelers) from 2022 to 2024?
SELECT
	maker,sum(electric_vehicles_sold) as sales_volume,
    quarter
from 
	ev_sales_by_makers em
join
	dim_date dd on dd.date = em.date
where 
	fiscal_year BETWEEN 2022 and 2024
    AND vehicle_category = '4-wheelers'
GROUP BY 
	maker,quarter
ORDER BY 
	maker , sales_volume desc;
    
    
--  How do the EV sales and penetration rates in Delhi compare to 
--  Karnataka for 2024? 


 SELECT 
	ss.state,sum(electric_vehicles_sold)as ev_sold,
    (SUM(ss.electric_vehicles_sold) / SUM(ss.total_vehicles_sold)) * 100 AS p_rate
 FROM 
	ev_sales_by_state ss
WHERE
	ss.state in ('Delhi','Karnataka')
GROUP BY
	ss.state;
	
    
-- List down the compounded annual growth rate (CAGR) in 4-wheeler 
-- units for the top 5 makers from 2022 to 2024. 


WITH sales_data AS (
    SELECT 
        se.maker,vehicle_category,
        dd.fiscal_year,
        SUM(se.electric_vehicles_sold) AS total_ev_sold
    FROM 
        ev_sales_by_makers se
    JOIN 
        dim_date dd ON dd.date = se.date
    WHERE 
        dd.fiscal_year BETWEEN 2022 and 2024
        and vehicle_category = '4-wheelers'
    GROUP BY 
        se.maker, dd.fiscal_year
	
),
cagr_calculation AS (
    SELECT 
        sd_2022.maker,
        sd_2022.total_ev_sold AS ev_sold_2022,
        sd_2024.total_ev_sold AS ev_sold_2024,
        (sd_2024.total_ev_sold / sd_2022.total_ev_sold)^1/2 - 1 AS cagr,
        DENSE_RANK() over(ORDER BY  (sd_2024.total_ev_sold / sd_2022.total_ev_sold)^1/3 - 1 desc) as rk
    FROM 
        sales_data sd_2022
    JOIN 
        sales_data sd_2024 ON sd_2022.maker = sd_2024.maker
    WHERE 
        sd_2022.fiscal_year = 2022 
        AND sd_2024.fiscal_year = 2024
)
SELECT 
	maker,
    ev_sold_2022, 
    ev_sold_2024, 
    cagr
FROM 
    cagr_calculation
where
	rk <= 5
ORDER BY 
    cagr DESC;


--  List down the top 10 states that had the highest compounded annual 
-- growth rate (CAGR) from 2022 to 2024 in total vehicles sold. 

WITH sales_data AS (
    SELECT 
        se.state,vehicle_category,
        dd.fiscal_year,
        SUM(se.electric_vehicles_sold) AS total_ev_sold
        
    FROM 
        ev_sales_by_state se
    JOIN 
        dim_date dd ON dd.date = se.date
    WHERE 
        dd.fiscal_year BETWEEN 2022 and 2024
        and vehicle_category = '4-wheelers'
    GROUP BY 
        se.state, dd.fiscal_year
	
),
cagr_calculation AS (
    SELECT 
        sd_2022.state,
        sd_2022.total_ev_sold AS ev_sold_2022,
        sd_2024.total_ev_sold AS ev_sold_2024,
        (sd_2024.total_ev_sold / sd_2022.total_ev_sold)^1/2 - 1 AS cagr,
        DENSE_RANK() over(ORDER BY  (sd_2024.total_ev_sold / sd_2022.total_ev_sold)^1/2 - 1 desc) as rk
    FROM 
        sales_data sd_2022
    JOIN 
        sales_data sd_2024 ON sd_2022.state = sd_2024.state
    WHERE 
        sd_2022.fiscal_year = 2022 
        AND sd_2024.fiscal_year = 2024
)
SELECT 
	state,
    ev_sold_2022, 
    ev_sold_2024, 
    cagr
FROM 
    cagr_calculation
where
	rk <= 10
ORDER BY 
    cagr DESC;



-- What is the projected number of EV sales (including 2-wheelers and 4
-- wheelers) for the top 10 states by penetration rate in 2030, based on the 
-- compounded annual growth rate (CAGR) from previous years?


WITH top_state as(
SELECT 
    se.state,cagrs.cagr, -- cagr = compound annual groth
    sum(electric_vehicles_sold)as ev_sold,
    sum(electric_vehicles_sold)*(1+cagr)^6 as 2030_projected_sales ,-- Projected Sales in 2030 = Total Sales in 2024×(1+CAGR)^6
    (SUM(se.electric_vehicles_sold) / SUM(se.total_vehicles_sold)) * 100 AS p_rate -- Penetration rate
    , DENSE_RANK() OVER(ORDER BY (SUM(se.electric_vehicles_sold) / SUM(se.total_vehicles_sold)) * 100 desc)as p_rank -- Penetration rank 
FROM 
    ev_sales_by_state se
JOIN 
    dim_date dd ON dd.date = se.date
join cagrs ON cagrs.state = se.state
WHERE 
    se.vehicle_category IN ('2-wheelers', '4-wheelers') 
    AND dd.fiscal_year = 2024 
GROUP BY 
    se.state
ORDER BY 
    p_rate DESC)

SELECT 
	state , p_rate,ev_sold, cagr,
    2030_projected_sales
FROM 
	top_state 
WHERE p_rank <=10;


-- Estimate the revenue growth rate of 4-wheeler and 2-wheelers 
-- EVs in India for 2022 vs 2024 and 2023 vs 2024, assuming an average 
-- unit price.


CREATE VIEW revanue_2022 as(                           -- Created view for 2022 revanue
SELECT es.state,sum(es.electric_vehicles_sold),es.vehicle_category,
case 
	WHEN es.vehicle_category = '2-wheelers' then sum(es.electric_vehicles_sold)*85000
    WHEN es.vehicle_category = '4-wheelers' then sum(es.electric_vehicles_sold)*1500000
    end as revanue
FROM
	ev_sales_by_state es
join dim_date dd on dd.date = es.date
WHERE
	dd.fiscal_year = 2022
GROUP BY 
	es.vehicle_category,es.state
ORDER BY
	revanue desc);
    
    
CREATE VIEW revanue_2023 AS
    (SELECT 
        es.state,
        SUM(es.electric_vehicles_sold),
        es.vehicle_category,
        CASE
            WHEN es.vehicle_category = '2-wheelers' THEN SUM(es.electric_vehicles_sold) * 85000
            WHEN es.vehicle_category = '4-wheelers' THEN SUM(es.electric_vehicles_sold) * 1500000
        END AS revanue
    FROM
        ev_sales_by_state es
            JOIN
        dim_date dd ON dd.date = es.date
    WHERE
        dd.fiscal_year = 2023
    GROUP BY es.vehicle_category , es.state
    ORDER BY revanue DESC);
    
    
CREATE VIEW revanue_2024 AS
    (SELECT 
        es.state,
        SUM(es.electric_vehicles_sold),
        es.vehicle_category,
        CASE
            WHEN es.vehicle_category = '2-wheelers' THEN SUM(es.electric_vehicles_sold) * 85000
            WHEN es.vehicle_category = '4-wheelers' THEN SUM(es.electric_vehicles_sold) * 1500000
        END AS revanue
    FROM
        ev_sales_by_state es
            JOIN
        dim_date dd ON dd.date = es.date
    WHERE
        dd.fiscal_year = 2024
    GROUP BY es.vehicle_category , es.state
    ORDER BY revanue DESC);


SELECT 
    (SUM(revanue_2024.revanue) - SUM(revanue_2022.revanue)) / (SUM(revanue_2022.revanue)) * 100 AS RGR_2022vs2024,
    (SUM(revanue_2024.revanue) - SUM(revanue_2023.revanue)) / SUM(revanue_2023.revanue) * 100 AS RGR_2023vs2024
FROM
    revanue_2022
        JOIN
    revanue_2023 ON revanue_2023.state = revanue_2022.state
        JOIN
    revanue_2024 ON revanue_2024.state = revanue_2023.state;



SELECT maker, sum(electric_vehicles_sold) 
from 
	ev_sales_by_makers
    
 GROUP BY maker
 ;
