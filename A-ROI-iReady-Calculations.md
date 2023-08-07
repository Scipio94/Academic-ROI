The SQL syntax below calculates the iReady diagnostic results of students from the original K-8 Tier 2 Cohort without replacement; meaning that if a student either achieves Tier 1 status or falls to Tier 3 they are excluded from the calculation in the next iReady diagnostic. This is done due to the academic team's focus on Tier 2 students. 

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

/*iReady 1 to iReady 2 Metrics*/ -- Cohort tracking students from iReady 1 to iReady 4 ELA 
SELECT 
 DISTINCT t1.ir1_Tier_Level,
  COUNT(*) OVER (PARTITION BY t1.ir1_Tier_Level) AS ir1_Count,
  COALESCE(t1.ir2_Tier_Level,'DNT') AS ir2_Tier_Level, -- Tier Level based on iReady 2 ORP
  COUNT(*) OVER (PARTITION BY t1.ir2_Tier_Level) AS ir2_Count,
  ROUND((COUNT(*) OVER (PARTITION BY t1.ir2_Tier_Level))/( COUNT(*) OVER (PARTITION BY t1.ir1_Tier_Level)),2) AS Pct -- Comparing Tier_Levels of Students who took both iReady 1 and iReady 2
FROM t1;

/*iReady1 to iReady 2 student list */ -- to be used for iReady 2 to iReady 3 metrics
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
WHERE t1.ir2_Tier_Level IS NOT NULL AND t1.ir2_Tier_Level = 'Tier 2') AS sub;--Filtering for students in Tier 2 as they were the academic focus

/*iReady 3 to iReady 4*/-- Will be used to calculate metrics for iReady 3 to iReady 4
WITH t2 AS -- Creating CTE for iReady 3 to 4 data
(SELECT 
  sub.student_id,
  sub.first_name,
  sub.last_name, 
  sub.student_grade,
  ir3_Tier_Level
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
WHERE sub.ir3_Tier_Level = 'Tier 2')


SELECT
  DISTINCT sub.ir3_Tier_Level,
  COUNT(*) OVER (PARTITION BY sub.ir3_Tier_Level) AS ir3_Count,
  COALESCE(sub.ir4_Tier_Level,'DNT') AS ir4_Tier_Level,
  COUNT(*) OVER (PARTITION BY sub.ir4_Tier_Level) AS ir4_Count,
  ROUND((COUNT(*) OVER (PARTITION BY sub.ir4_Tier_Level))/ (COUNT(*) OVER (PARTITION BY sub.ir3_Tier_Level)),2) AS Pct
FROM
(SELECT
  t2.ir3_Tier_Level,
  CASE -- Creating iReady Tiers
    WHEN ir4.Overall_Relative_Placement IN ('Early On Grade Level', 'Mid or Above Grade Level') THEN 'Tier 1'
    WHEN ir4.Overall_Relative_Placement = '1 Grade Level Below' THEN 'Tier 2'
    WHEN ir4.Overall_Relative_Placement IN ('2 Grade Levels Below', '3 or More Grade Levels Below') THEN 'Tier 3'
    END AS ir4_Tier_Level,
FROM t2
LEFT JOIN `my-data-project-36654.iReady_SY_2223.iReady_4_ELA` AS ir4 
USING(Student_ID)) AS sub;


SELECT
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
LEFT JOIN `my-data-project-36654.iReady_SY_2223.iReady_4_ELA` AS ir4 
ON sub.Student_ID = ir4.Student_ID
WHERE sub.ir3_Tier_Level IS NOT NULL AND sub.ir3_Tier_Level = 'Tier 2') AS sub1;

/*Tier 2 K-8 Cohort*/
SELECT
  DISTINCT Student_Grade,
  COUNT(*) OVER (PARTITION BY Student_Grade) AS Count_,
  COUNT(*) OVER () AS Total
FROM `my-data-project-36654.iReady_SY_2223.iReady_ELA_1` 
WHERE Overall_Relative_Placement = '1 Grade Level Below' AND Student_Grade NOT IN ('9','10','11','12')
ORDER BY Count_ DESC;

/*Tier 2 K-8 Cohort Percentage of Student Population*/
SELECT
  DISTINCT Overall_Relative_Placement,
  COUNT(*) OVER (PARTITION BY Overall_Relative_Placement) AS Count_,
  COUNT(*) OVER () AS Total,
  ROUND((COUNT(*) OVER (PARTITION BY Overall_Relative_Placement))/ (COUNT(*) OVER ()),2) AS Pct
FROM `my-data-project-36654.iReady_SY_2223.iReady_ELA_1` 
WHERE Student_Grade NOT IN ('9','10','11','12')

~~~
