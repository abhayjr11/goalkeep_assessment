Exercise 2 SQL Query Optimization

https://colab.research.google.com/drive/1bNtVitGQHewiCA1ElvxNHXQQBYcPjNFx?usp=sharing

Note : The data in the dataset is of july 2025 so getting zero records on 90 days condition so i am using 
400 days instead of 90 days.


#1. Review and identify inefficiencies

1: Syntax mistake in line 4 (the comma should not be there)
         COUNT(*) AS num_complaints,

2: This query is reading data from same table 3 times(Unnecessary) which generally slows down the performance.

3: unnecessary Self-Join two times
     1:  Self-join, which is not needed is used (recent table)
          JOIN (
             SELECT *
             FROM service_requests
             WHERE created_date >= CURRENT_DATE - INTERVAL '90 days'
              ) AS recent
           ON sr.unique_key = recent.unique_key
         --> we already can get the same result applying where condition instead using a self-join.

     2:  Self-join, which is not needed is used (zip_filter)
          JOIN (
            SELECT DISTINCT zip_code
            FROM service_requests
            WHERE zip_code IS NOT NULL
            ) AS zip_filter
         ON sr.zip_code = zip_filter.zip_code
          --> this join is also not necessary no row id affected by this join, later we are going to use same condition
           which this join is applying 

4: Using Select * from 
      selecting all rows which are not even needed will decrease the performance of query.
      also we can filter the records earlier before joining the tables is best practice which also increase the query perfromance 

#2. Rewrite the query for improved readability and performance

select distinct zip_code, agency, count(*)  as num_complaints from service_requests
WHERE complaint_type = 'Noise - Residential'
and created_date >= CURRENT_DATE - INTERVAL '400 days'
and closed_date IS NOT NULL
and zip_code IS NOT NULL
group by 1,2
order by num_complaints  DESC
limit 10;

#3. Use CTEs to break query logic into smaller chunks

with cte_com_type as (
select zip_code, agency,  created_date, closed_date
from service_requests
WHERE complaint_type = 'Noise - Residential'

),

cte_within_time_interval as (
select zip_code, agency, created_date, closed_date
from cte_com_type
where created_date >= CURRENT_DATE - INTERVAL '400 days'
),

select zip_code, agency, count(*) as num_complaints
from cte_within_time_interval
where closed_date IS NOT NULL and zip_code IS NOT NULL
group by zip_code, agency
order by num_complaints DESC
limit 10;

#4. Note: the output of the optimized query should remain the same

Output is remains same for all two query i wrote
<img width="336" height="307" alt="image" src="https://github.com/user-attachments/assets/3c04f6e9-0fbd-48f5-bd88-d8095992e040" />



