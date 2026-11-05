# SQL Job Market Analysis: Data Analyst Roles

An SQL project that digs into real job-posting data to answer one practical question: **what should an aspiring data analyst learn to land a high-paying, in-demand role?**

I wrote five queries that move from *"which jobs pay the most?"* to *"which skills are worth learning first?"*, using joins, CTEs, aggregations and filtering on a relational job-postings database.

---

## Background

The data analyst job market is crowded, and it isn't obvious which skills actually pay off. I built this project to find out which roles pay the most, which skills those roles ask for, which skills employers want most often, and where high pay and high demand overlap.

## Questions I Answered

1. What are the top-paying Data Analyst jobs (remote or in India)?
2. What skills do those top-paying jobs require?
3. What are the most in-demand skills for Data Analysts?
4. Which skills are associated with the highest average salaries?
5. What are the most *optimal* skills to learn (high demand **and** high salary)?

## Tools Used

- **SQL** for all querying and analysis
- **PostgreSQL** as the database
- **VS Code** as the editor
- **Git & GitHub** for version control

## Dataset

The project uses a job-postings dataset stored in four related PostgreSQL tables: one fact table, two dimension tables and one bridge table that links jobs to skills.

### Table relationships

- `job_postings_fact.company_id` → `company_dim.company_id`
- `skills_job_dim.job_id` → `job_postings_fact.job_id`
- `skills_job_dim.skill_id` → `skills_dim.skill_id`

`skills_job_dim` uses a composite primary key (`job_id`, `skill_id`), which makes the relationship between jobs and skills many-to-many.

### `job_postings_fact`: one row per job posting

| Column | Type | Notes |
|---|---|---|
| `job_id` | INTEGER | Primary key |
| `company_id` | INTEGER | Foreign key to `company_dim` |
| `job_title_short` | VARCHAR(255) | Standardized role, e.g. "Data Analyst" |
| `job_title` | TEXT | Full title as posted |
| `job_location` | TEXT | Location of the job |
| `job_via` | TEXT | Platform the job was posted on |
| `job_schedule_type` | TEXT | Full-time, contract, etc. |
| `job_work_from_home` | BOOLEAN | Whether the job is remote |
| `search_location` | TEXT | Location used when the posting was collected |
| `job_posted_date` | TIMESTAMP | When the job was posted |
| `job_no_degree_mention` | BOOLEAN | Whether the posting mentions no degree requirement |
| `job_health_insurance` | BOOLEAN | Whether health insurance is mentioned |
| `job_country` | TEXT | Country of the job |
| `salary_rate` | TEXT | Pay period of the listed salary (year, hour, etc.) |
| `salary_year_avg` | Numeric | Average yearly salary |
| `salary_hour_avg` | Numeric | Average hourly salary |

### `company_dim`: company details

| Column | Type | Notes |
|---|---|---|
| `company_id` | INTEGER | Primary key |
| `name` | TEXT | Company name |
| `link` | TEXT | Company link |
| `link_google` | TEXT | Google search link for the company |
| `thumbnail` | TEXT | Company logo or thumbnail |

### `skills_dim`: list of skills

| Column | Type | Notes |
|---|---|---|
| `skill_id` | INTEGER | Primary key |
| `skills` | TEXT | Skill name, e.g. "sql" |
| `type` | TEXT | Skill category |

### `skills_job_dim`: bridge table linking jobs to skills

| Column | Type | Notes |
|---|---|---|
| `job_id` | INTEGER | Foreign key to `job_postings_fact` (part of primary key) |
| `skill_id` | INTEGER | Foreign key to `skills_dim` (part of primary key) |

## Project Structure

```
SQL_job_analysis_data_project/
├── sql_load/    # Scripts to create the database and tables and load the data
├── Query/       # The five analysis queries (one file per question)
├── .gitignore
└── README.md
```

---

## The Analysis

### 1. Top-Paying Data Analyst Jobs

To find the best-paid opportunities, I filtered for Data Analyst roles that are remote (`Anywhere`) or located in `India`, kept only postings with a listed salary, and joined to `company_dim` for the employer name. The top 10 by average yearly salary are returned.

```sql
SELECT
    job_id,
    job_title,
    job_location,
    job_schedule_type,
    salary_year_avg,
    job_posted_date,
    name AS company_name
FROM
    job_postings_fact
LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
WHERE
    job_title_short = 'Data Analyst' AND
    job_location IN ('Anywhere', 'India') AND
    salary_year_avg IS NOT NULL
ORDER BY
    salary_year_avg DESC
LIMIT 10;
```

| Job title | Company | Avg yearly salary |
|---|---|---|
| Data Analyst | Mantys | $650,000 |
| Director of Analytics | Meta | $336,500 |
| Associate Director, Data Insights | AT&T | $255,830 |
| Data Analyst, Marketing | Pinterest Job Advertisements | $232,423 |
| Data Analyst (Hybrid/Remote) | Uclahealthcareers | $217,000 |
| Principal Data Analyst (Remote) | SmartAsset | $205,000 |
| Director, Data Analyst - HYBRID | Inclusively | $189,309 |
| Principal Data Analyst, AV Performance | Motional | $189,000 |
| Principal Data Analyst | SmartAsset | $186,000 |
| ERM Data Analyst | Get It Recruit - Information Technology | $184,000 |

**Insights**
- Salaries in the top 10 range from **$184K to $650K**.
- The top posting (Mantys, $650K) is almost double the next one, so it is likely an outlier or a data-entry quirk and should be read with caution.
- Seniority drives pay. Most high-paying roles are Director, Associate Director or Principal level.
- Every result is a **full-time, remote (`Anywhere`)** role. No India-based posting made the top 10.
- Employers range from large companies (Meta, AT&T, Pinterest) to smaller firms and recruiters.

---

### 2. Skills Required for the Top-Paying Jobs

I reused the query above as a CTE (`top_paying_jobs`) and joined it to `skills_job_dim` and `skills_dim` to see which skills those roles list.

```sql
WITH top_paying_jobs AS (
    SELECT
        job_id,
        job_title,
        salary_year_avg,
        name AS company_name
    FROM
        job_postings_fact
    LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
    WHERE
        job_title_short = 'Data Analyst' AND
        job_location IN ('India', 'Anywhere') AND
        salary_year_avg IS NOT NULL
    ORDER BY
        salary_year_avg DESC
    LIMIT 10
)

SELECT
    top_paying_jobs.*,
    skills
FROM top_paying_jobs
INNER JOIN skills_job_dim ON top_paying_jobs.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
ORDER BY
    salary_year_avg DESC;
```

**Insights**
- **SQL, Python and Tableau** recur most often across the top-paying roles, with **R** also common.
- The best-paid listing with skills, AT&T's Associate Director role ($255K), asks for a broad stack: SQL, Python, R, cloud and big-data tools (Azure, AWS, Databricks, PySpark), Pandas, Jupyter, and BI tools (Tableau, Power BI, Excel, PowerPoint).
- Other high-paying roles mix core analytics skills with niche tools such as Hadoop, Snowflake, Oracle and Go.
- The two highest-paying jobs (Mantys and Meta) do not appear in the output. They have no skills listed in the data, so the `INNER JOIN` drops them.

---

### 3. Most In-Demand Skills for Data Analysts

I counted how many Data Analyst postings mention each skill and returned the top 5.

```sql
SELECT
    skills,
    COUNT(skills_job_dim.job_id) AS demand_count
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
GROUP BY
    skills
ORDER BY
    demand_count DESC
LIMIT 5;
```

| Skill | Postings |
|---|---|
| SQL | 92,628 |
| Excel | 67,031 |
| Python | 57,326 |
| Tableau | 46,554 |
| Power BI | 39,468 |

**Insights**
- **SQL is the most requested skill**, ahead of Excel by about 25,600 postings.
- The top five are a mix of **querying (SQL), spreadsheets (Excel), programming (Python) and visualization (Tableau, Power BI)**. These are the fundamentals of the role.

---

### 4. Top Skills by Average Salary

For remote Data Analyst postings with a listed salary, I calculated the average salary linked to each skill and returned the top 25.

```sql
SELECT
    skills,
    ROUND(AVG(salary_year_avg), 0) AS avg_salary
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
    AND salary_year_avg IS NOT NULL
    AND job_work_from_home = True
GROUP BY
    skills
ORDER BY
    avg_salary DESC
LIMIT 25;
```

| Skill | Avg salary |
|---|---|
| PySpark | $208,172 |
| Bitbucket | $189,155 |
| Couchbase | $160,515 |
| Watson | $160,515 |
| DataRobot | $155,486 |
| GitLab | $154,500 |
| Swift | $153,750 |
| Jupyter | $152,777 |
| Pandas | $151,821 |
| Elasticsearch | $145,000 |

*(Top 10 of 25 shown; the rest run down to MicroStrategy at $121,619.)*

**Insights**
- The highest-paying skills are mostly **big-data, machine-learning and engineering tools**: PySpark, Databricks, Airflow, Kubernetes, Scala, scikit-learn.
- **Python data libraries** (Pandas, NumPy, Jupyter, scikit-learn) all sit in the top 25, so going beyond basic Python pays off.
- **Version control and collaboration tools** (Bitbucket, GitLab, Jenkins, Atlassian, Notion) also appear, which suggests employers pay more for analysts who work like engineers.
- This query has no minimum-demand filter, so some averages (for example Couchbase and Watson, which share the same value) probably come from very few postings. That is why Query 5 adds a threshold.

---

### 5. Most Optimal Skills to Learn

The final query combines demand and salary. It looks at remote Data Analyst roles with a listed salary and keeps only skills that appear in **more than 10 postings**, so the averages are more reliable. Results are sorted by average salary, then by demand.

```sql
SELECT
    skills_dim.skill_id,
    skills_dim.skills,
    COUNT(skills_job_dim.job_id) AS demand_count,
    ROUND(AVG(job_postings_fact.salary_year_avg), 0) AS avg_salary
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
    AND salary_year_avg IS NOT NULL
    AND job_work_from_home = True
GROUP BY
    skills_dim.skill_id
HAVING
    COUNT(skills_job_dim.job_id) > 10
ORDER BY
    avg_salary DESC,
    demand_count DESC
LIMIT 25;
```

| Skill | Demand (postings) | Avg salary |
|---|---|---|
| Go | 27 | $115,320 |
| Hadoop | 22 | $113,193 |
| Snowflake | 37 | $112,948 |
| Azure | 34 | $111,225 |
| AWS | 32 | $108,317 |
| Oracle | 37 | $104,534 |
| Looker | 49 | $103,795 |
| **Python** | **236** | **$101,397** |
| R | 148 | $100,499 |
| **Tableau** | **230** | **$99,288** |

*(Selected rows from the top 25.)*

**Insights**
- **Python (236 postings, ~$101K) and Tableau (230 postings, ~$99K)** have by far the highest demand in this list while still paying around $100K, which makes them the safest high-value skills.
- **Cloud and data-warehouse skills** (Snowflake, Azure, AWS) pay roughly $108K–$113K with solid demand, a clear premium over visualization tools alone.
- **Go and Hadoop** pay the most among skills with real demand, but they appear less often (22–27 postings), so they are more niche bets.
- **R (148 postings, ~$100K)** is still a strong secondary language after Python.

---

## Key Takeaways

1. **Start with the fundamentals.** SQL, Excel, Python and Tableau or Power BI are the most requested skills (Query 3).
2. **Python is the best value skill.** It combines very high demand with an above-average salary (Query 5).
3. **Add cloud and big-data skills to raise your pay.** Snowflake, Azure, AWS, Hadoop and PySpark are associated with the highest salaries (Queries 4 and 5).
4. **Seniority matters as much as skills.** The top-paying roles are mostly Director and Principal level (Query 1).
5. **Most top-paying roles are remote.** Remote roles dominate the high end of the salary range (Query 1).

## Notes & Limitations

- Salary figures are `salary_year_avg` and only cover postings that list a salary, so they may not represent the whole market.
- Query 1's top result ($650K) looks like an outlier.
- Queries 4 and 5 cover **remote roles only** (`job_work_from_home = True`), while Query 3 covers all Data Analyst postings.
- `sas` appears twice in the Query 5 output because it has two entries in `skills_dim` (skill IDs 7 and 186) and the query groups by `skill_id`. Grouping by the skill name instead would merge them.
- Query 2 uses an `INNER JOIN`, so top-paying jobs with no listed skills are excluded.

## What I Learned

- Writing multi-table queries with `INNER JOIN` and `LEFT JOIN` across fact, dimension and bridge tables
- Using **CTEs** to build on a previous query
- Aggregating with `COUNT`, `AVG` and `ROUND`, and filtering groups with `HAVING`
- Handling `NULL` values and working with boolean filters
- Turning raw query output into insights that answer a real question
