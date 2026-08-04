# Exercise 2 - SQL Query Optimization

**Notebook:**  
https://colab.research.google.com/drive/1bNtVitGQHewiCA1ElvxNHXQQBYcPjNFx?usp=sharing

> **Note**
>
> The dataset contains data only until **July 2025**. Therefore, using:
>
> ```sql
> CURRENT_DATE - INTERVAL '90 days'
> ```
>
> returns **0 records**. For demonstration purposes, I used:
>
> ```sql
> CURRENT_DATE - INTERVAL '400 days'
> ```
>
> so that the query produces meaningful results.

---

# 1. Review and Identify Inefficiencies

## 1. Syntax Error

```sql
COUNT(*) AS num_complaints,
```

**Issue**

There is an extra comma after `COUNT(*) AS num_complaints`.

---

## 2. Multiple Table Scans

The query reads the **same table three times**, which increases I/O and slows down execution.

```sql
FROM (
    SELECT *
    FROM service_requests
    WHERE complaint_type = 'Noise - Residential'
) AS sr

JOIN (
    SELECT *
    FROM service_requests
    WHERE created_date >= CURRENT_DATE - INTERVAL '90 days'
) AS recent

JOIN (
    SELECT DISTINCT zip_code
    FROM service_requests
    WHERE zip_code IS NOT NULL
) AS zip_filter
```

**Issue**

- Reads the same table three times.
- Performs unnecessary work.
- Higher execution cost.

---

## 3. Unnecessary Self Join (`recent`)

```sql
JOIN (
    SELECT *
    FROM service_requests
    WHERE created_date >= CURRENT_DATE - INTERVAL '90 days'
) AS recent
ON sr.unique_key = recent.unique_key
```

**Issue**

This self-join is unnecessary.

The same result can be achieved by applying the filter directly:

```sql
WHERE created_date >= CURRENT_DATE - INTERVAL '90 days'
```

This removes an entire table scan.

---

## 4. Unnecessary Self Join (`zip_filter`)

```sql
JOIN (
    SELECT DISTINCT zip_code
    FROM service_requests
    WHERE zip_code IS NOT NULL
) AS zip_filter
ON sr.zip_code = zip_filter.zip_code
```

**Issue**

This join does not filter any additional rows because later the query already applies:

```sql
WHERE zip_code IS NOT NULL
```

Therefore, this join performs unnecessary work and should be removed.

---

## 5. Using `SELECT *`

```sql
SELECT *
```

**Issue**

Using `SELECT *` retrieves every column, even though only a few are needed.

It is better to select only the required columns:

```sql
SELECT
    zip_code,
    agency,
    created_date,
    closed_date
```

This reduces memory usage and improves performance.

---

# 2. Rewrite the Query for Better Performance

```sql
SELECT
    zip_code,
    agency,
    COUNT(*) AS num_complaints
FROM service_requests
WHERE complaint_type = 'Noise - Residential'
    AND created_date >= CURRENT_DATE - INTERVAL '400 days'
    AND closed_date IS NOT NULL
    AND zip_code IS NOT NULL
GROUP BY
    zip_code,
    agency
ORDER BY
    num_complaints DESC
LIMIT 10;
```

### Improvements

- ✅ Single table scan
- ✅ Removed unnecessary joins
- ✅ Early filtering
- ✅ Better readability
- ✅ Lower execution cost

---

# 3. Rewrite Using CTEs

```sql
WITH cte_com_type AS (

    SELECT
        zip_code,
        agency,
        created_date,
        closed_date
    FROM service_requests
    WHERE complaint_type = 'Noise - Residential'

),

cte_within_time_interval AS (

    SELECT
        zip_code,
        agency,
        created_date,
        closed_date
    FROM cte_com_type
    WHERE created_date >= CURRENT_DATE - INTERVAL '400 days'

)

SELECT
    zip_code,
    agency,
    COUNT(*) AS num_complaints
FROM cte_within_time_interval
WHERE closed_date IS NOT NULL
    AND zip_code IS NOT NULL
GROUP BY
    zip_code,
    agency
ORDER BY
    num_complaints DESC
LIMIT 10;
```

### Why use CTEs?

- Improves readability.
- Breaks complex logic into smaller steps.
- Easier to debug.
- Easier to maintain.
- Modern query optimizers usually generate an efficient execution plan for simple CTEs.

---

# 4. Output Validation

Both the optimized query and the CTE version return the **same result**, confirming that the optimization does not change the output.

<p align="center">
<img src="https://github.com/user-attachments/assets/3c04f6e9-0fbd-48f5-bd88-d8095992e040" width="350">
</p>

---

# Summary

| Improvement | Status |
|-------------|--------|
| Fixed syntax error | ✅ |
| Removed unnecessary self-joins | ✅ |
| Reduced table scans (3 → 1) | ✅ |
| Removed `SELECT *` | ✅ |
| Applied filters earlier | ✅ |
| Improved readability | ✅ |
| Used CTEs for modular query design | ✅ |
| Output remains unchanged | ✅ |
