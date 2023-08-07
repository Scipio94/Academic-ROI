The SQL syntax below combines all the calculations from each iReady diagnostic via UNION ALL statements for aggregate calculations:

~~~ SQL
CREATE TEMP TABLE t1 AS
SELECT 
  ir1.First_Name,
  ir1.Last_Name,
  ir1.Student_Grade,
  ir1.Overall_Relative_Placement AS ir1_ORP,
  CASE -- Creating iReady Tiers
    WHEN ir1.Overall_Relative_Placement IN ('Early On Grade Level', 'Mid or Above Grade Level') THEN 'Tier 1'
    WHEN ir1.Overall_Relative_Placement = '1 Grade Level Below' THEN 'Tier 2'
    WHEN ir1.Overall_Relative_Placement IN ('2 Grade Levels Below', '3 or More Grade Levels Below') THEN 'Tier 3'
    END AS ir1_Tier_Level,
  ir2.Overall_Relative_Placement AS ir2_ORP,
   CASE -- Creating iReady Tiers
    WHEN ir2.Overall_Relative_Placement IN ('Early On Grade Level', 'Mid or Above Grade Level') THEN 'Tier 1'
    WHEN ir2.Overall_Relative_Placement = '1 Grade Level Below' THEN 'Tier 2'
    WHEN ir2.Overall_Relative_Placement IN ('2 Grade Levels Below', '3 or More Grade Levels Below') THEN 'Tier 3'
    END AS ir2_Tier_Level,
  ir1.Student_ID
FROM `my-data-project-36654.iReady_SY_2223.iReady_ELA_1` as ir1 -- iReady 1 ELA
LEFT JOIN `my-data-project-36654.iReady_SY_2223.iReady_2_ELA` AS ir2
USING(Student_ID)
WHERE ir1.Student_Grade NOT IN ('9','10','11','12') AND ir1.Overall_Relative_Placement = '1 Grade Level Below';

WITH ir1_2 AS -- Creating CTE with all calculation
(/*iReady 1 to iReady 2 Metrics*/
SELECT 
 DISTINCT t1.ir1_Tier_Level,
  COUNT(*) OVER (PARTITION BY t1.ir1_Tier_Level) AS ir1_Count,
  COALESCE(t1.ir2_Tier_Level,'DNT') AS ir2_Tier_Level, -- Tier Level based on iReady 2 ORP
  COUNT(*) OVER (PARTITION BY t1.ir2_Tier_Level) AS ir2_Count,
  ROUND((COUNT(*) OVER (PARTITION BY t1.ir2_Tier_Level))/( COUNT(*) OVER (PARTITION BY t1.ir1_Tier_Level)),2) AS Pct 
FROM t1),

ir2_3 AS 
(/*iReady1 to iReady 2 student list */ -- to be used for iReady 2 to iReady 3 metrics
SELECT 
  DISTINCT sub.ir2_Tier_Level,
  COUNT(*) OVER (PARTITION BY sub.ir2_Tier_Level) AS ir2_Count,
  COALESCE(sub.ir3_Tier_Level, 'DNT') AS ir3_Tier_Level,
  COUNT(*) OVER (PARTITION BY sub.ir3_Tier_Level) AS ir3_Count,
  ROUND((COUNT(*) OVER (PARTITION BY sub.ir3_Tier_Level) )/ (COUNT(*) OVER (PARTITION BY sub.ir2_Tier_Level) ),2) AS Pct
FROM
(SELECT 
 DISTINCT t1.Student_ID,
  t1.First_Name,
  t1.Last_Name,
  t1.Student_Grade,
  t1.ir1_Tier_Level,
  t1.ir1_ORP,
  t1.ir2_Tier_Level,
  t1.ir2_ORP,
   CASE -- Creating iReady Tiers
    WHEN ir3.Overall_Relative_Placement IN ('Early On Grade Level', 'Mid or Above Grade Level') THEN 'Tier 1'
    WHEN ir3.Overall_Relative_Placement = '1 Grade Level Below' THEN 'Tier 2'
    WHEN ir3.Overall_Relative_Placement IN ('2 Grade Levels Below', '3 or More Grade Levels Below') THEN 'Tier 3'
    END AS ir3_Tier_Level,
  ir3.Overall_Relative_Placement AS ir3_ORP
FROM t1
LEFT JOIN `my-data-project-36654.iReady_SY_2223.iReady_3_ELA` AS ir3-- iReady 3 ELA
ON t1.Student_ID = ir3.Student_ID
WHERE t1.ir2_Tier_Level IS NOT NULL AND t1.ir2_Tier_Level = 'Tier 2') AS sub),

ir3_4 AS
(SELECT
  DISTINCT sub1.ir3_Tier_Level,
  COUNT(*) OVER (PARTITION BY sub1.ir3_Tier_Level) AS ir3_Count,
  COALESCE(sub1.ir4_Tier_Level,'DNT') AS ir4_Tier_Level,
  COUNT(*) OVER (PARTITION BY sub1.ir4_Tier_Level) AS ir4_Count,
  ROUND((COUNT(*) OVER (PARTITION BY sub1.ir4_Tier_Level))/ (COUNT(*) OVER (PARTITION BY sub1.ir3_Tier_Level)),2) AS Pct
FROM
(SELECT 
  sub.student_id,
  sub.first_name,
  sub.last_name, 
  sub.student_grade,
  ir3_Tier_Level,
  ir4.Overall_Relative_Placement,
  CASE
    WHEN ir4.Overall_Relative_Placement IN ('Early On Grade Level', 'Mid or Above Grade Level') THEN 'Tier 1'
    WHEN ir4.Overall_Relative_Placement = '1 Grade Level Below' THEN 'Tier 2'
    WHEN ir4.Overall_Relative_Placement IN ('2 Grade Levels Below', '3 or More Grade Levels Below') THEN 'Tier 3'
    END AS ir4_Tier_Level
FROM
(SELECT 
 DISTINCT t1.Student_ID,
  t1.First_Name,
  t1.Last_Name,
  t1.Student_Grade,
  t1.ir1_Tier_Level,
  t1.ir1_ORP,
  t1.ir2_Tier_Level,
  t1.ir2_ORP,
   CASE -- Creating iReady Tiers
    WHEN ir3.Overall_Relative_Placement IN ('Early On Grade Level', 'Mid or Above Grade Level') THEN 'Tier 1'
    WHEN ir3.Overall_Relative_Placement = '1 Grade Level Below' THEN 'Tier 2'
    WHEN ir3.Overall_Relative_Placement IN ('2 Grade Levels Below', '3 or More Grade Levels Below') THEN 'Tier 3'
    END AS ir3_Tier_Level,
  ir3.Overall_Relative_Placement AS ir3_ORP
FROM t1
LEFT JOIN `my-data-project-36654.iReady_SY_2223.iReady_3_ELA` AS ir3-- iReady 3 ELA
ON t1.Student_ID = ir3.Student_ID
WHERE t1.ir2_Tier_Level IS NOT NULL AND t1.ir2_Tier_Level = 'Tier 2') AS sub
LEFT JOIN `my-data-project-36654.iReady_SY_2223.iReady_4_ELA` AS ir4 -- iReady 4 
USING(Student_ID)
WHERE sub.ir3_Tier_Level IS NOT NULL AND sub.ir3_Tier_Level = 'Tier 2') AS sub1)


/*Tier Aggreagtions*/
SELECT
  DISTINCT sub.Tier,
  SUM(sub.Count_) OVER (PARTITION BY sub.Tier) AS Total,
  ROUND((SUM(sub.Count_) OVER (PARTITION BY sub.Tier))/ 283,2) AS Pct --K-8 iReady 1 Tier 2 Cohort
FROM 
(SELECT -- Combining all results via Union for Aggregation
  ir1_2.ir2_Tier_Level AS Tier,
  ir1_2.ir2_Count AS Count_
FROM ir1_2
UNION ALL
SELECT
  ir2_3.ir3_Tier_Level AS Tier,
  ir2_3.ir3_Count AS Count_
FROM ir2_3
UNION ALL
SELECT 
  ir3_4.ir4_Tier_Level AS Tier,
  ir3_4.ir4_Count AS Count_
FROM ir3_4) AS sub
WHERE sub.Tier = 'Tier 1' -- Filtering for students 
ORDER BY Pct DESC;

/*Tier 2 Calculation*/
SELECT
  283 - (222 + 21 + 5) AS Tier_2_Count,
  ROUND((283 - (222 + 21 + 5)) / 283,2) AS Tier_2_Pct;

/*Cost per student*/
SELECT
  ROUND(20757.50/ 283,2) AS Cost_Per_Student,-- Cost of iReady/ number of students in Cohort
  ROUND(78/(20757.50/ 283),2) AS A_ROI -- Percentage of students that achieve Tier 1 status/ Avg cost per student

-- Cost of iReady resource $20,757.50
-- For every $ spent there is an expected increase of 1.06%  in the percentage of students in Tier 1

-- 222 Students in K-8 who began the year in Tier 2 based on their initial iReady Diagnostic moved into Tier 1
-- 35 Students remained in Tier 2
-- 21 Students in K-8 who began the year in Tier 2 based on their initial iReady Diagnostic moved into Tier 3
-- 5 students in K-8 who did not take all four iReady diagnostics
-- Cohort consisted of 284 students

~~~
