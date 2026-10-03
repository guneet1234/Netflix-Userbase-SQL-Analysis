# Netflix-Userbase-SQL-Analysis

1) Project Overview
   
   This project analyses a Netflix user base dataset of 2,500 users using SQL. The data covers each user's subscription plan, monthly revenue, country, age, gender, device, join date and last monthly payment date. The goal is to understand who the users are and how they differ across plans, countries and devices. I wrote queries to answer 6 questions, such as how age group relates to plan choice, which countries bring in the most revenue, which devices people prefer, and how the user mix changed by join year.
Tool used: SQL

2) Dataset Explanation
   
The dataset comes as one Excel file with 5 sheets: a main user table and four small lookup tables that give names to the ID columns.
2,500 users across 10 countries
3 subscription plans and 4 device types
Users joined between Sep 2021 and Jun 2023
Ages range from 26 to 51

Main table: NetflixUserbase
User_ID: unique ID for each user
Subscription_ID: the plan the user is on
Monthly_Revenue: amount paid per month (10 to 15)
Join_Date: date the user joined
LMP: last monthly payment date
Country_Id: the user's country
Age: the user's age
Gender_Id: the user's gender
Device_Id: the device the user watches on

Lookup tables
Subscription table (linked through Subscription_ID): Basic, Standard, Premium
Country table (linked through Country_Id): United States, Canada, United Kingdom, Australia, Germany, France, Brazil, Mexico, Spain, Italy
Gender table (linked through Gender_Id): Male, Female
Device table (linked through Device_Id): Smartphone, Tablet, Smart TV, Laptop

The main table only stores IDs, so I joined it with the lookup tables whenever I needed the actual plan, country, gender or device names.

3) Data Understanding and Cleaning
   
The main table holds all the user information and connects to the lookup tables through ID columns. LMP has no full form in the dataset, so I treated it as "last monthly payment". There is no total revenue column, so I calculated it myself.I checked for missing values, duplicate User_IDs and unmatched IDs, and found none, so no cleaning was needed.

4) Analytical Question

### Question 1

**Question:** What is the total revenue each user has paid so far? Create a new column Total_Revenue by multiplying each user's Monthly_Revenue by the number of months from Join_Date to LMP.

**Code:**

```sql
SELECT User_ID, Monthly_Revenue, Join_Date, LMP,
       Monthly_Revenue * (TIMESTAMPDIFF(MONTH, Join_Date, LMP) + 1) AS Total_Revenue
FROM NetflixUserbase;
```

**Output:**

```
User_ID  Monthly_Revenue  Join_Date   LMP         Total_Revenue
1        10               2022-01-15  2023-06-10  170
2        15               2021-09-05  2023-06-22  330
3        12               2023-02-28  2023-06-27  48
4        12               2022-07-10  2023-06-26  144
5        10               2023-05-01  2023-06-28  20
6        15               2022-03-18  2023-06-27  240
7        12               2021-12-09  2023-06-25  228
8        10               2023-04-02  2023-06-24  30
9        12               2022-10-20  2023-06-23  108
10       15               2023-01-07  2023-06-22  90
```

### Question 2

**Question:** Which subscription plan do different age groups prefer? Group users into age groups (Under 30, 30-39, 40-49, 50+), then count the users on each plan in every group and show the total revenue they brought in.

**Code:**

```sql
SELECT
CASE
WHEN n.Age < 30 THEN 'Under 30'
WHEN n.Age < 40 THEN '30-39'
WHEN n.Age < 50 THEN '40-49'
ELSE '50+'
END AS Age_Group,
s.Subscription_Type,
COUNT(*) AS Users,
SUM(n.Monthly_Revenue * (TIMESTAMPDIFF(MONTH, n.Join_Date, n.LMP) + 1)) AS Total_Revenue
FROM NetflixUserbase n
JOIN Subscription_table s ON n.Subscription_ID = s.Subscription_code
GROUP BY Age_Group, s.Subscription_Type
ORDER BY Age_Group, Users DESC;
```

**Output:**

```
Age_Group  Subscription_Type  Users  Total_Revenue
30-39      Basic              419    55514
30-39      Standard           308    41179
30-39      Premium            293    39002
40-49      Basic              388    50578
40-49      Standard           318    43538
40-49      Premium            290    39024
50+        Basic              69     9344
50+        Standard           56     7302
50+        Premium            52     7118
Under 30   Basic              123    17181
Under 30   Premium            98     13093
Under 30   Standard           86     11103
```

**Analysis:** The Basic plan is the most used plan and it generates the most revenue. Its target audience is the 30-39 age group. If the under 30 group was also targeted, the revenue would increase.
```
### Question 3

**Question:** What is the gap between the most and least selling subscription plan? Count the users on each plan and show the total revenue each plan brought in.

**Code:**

```sql
SELECT
Subscription_table.Subscription_Type,
COUNT(*) AS Users,
SUM(NetflixUserbase.Monthly_Revenue *
(TIMESTAMPDIFF(MONTH, NetflixUserbase.Join_Date, NetflixUserbase.LMP)
+ 1)) AS Total_Revenue
FROM NetflixUserbase
JOIN Subscription_table
ON NetflixUserbase.Subscription_ID = Subscription_table.Subscription_code
GROUP BY Subscription_table.Subscription_Type
ORDER BY Users DESC;
```

**Output:**

```
Subscription_Type  Users  Total_Revenue
Basic              999    132617
Standard           768    103122
Premium            733    98237
```

**Analysis:** Basic is the most selling plan and generates the most revenue (132,617 from 999 users), while Premium is the least selling (98,237 from 733 users). The gap between Basic and Premium is 266 users and 34,380 in revenue. There is not much gap between Standard and Premium, only 35 users.
```
### Question 4

**Question:** Which countries generate the highest and lowest revenue on each plan? Show the total revenue and the average revenue per user for every country and plan, followed by the total revenue and average revenue of that country.

**Code:**

```sql
SELECT
Country,
IFNULL(Plan_Name, 'Total') AS Subscription_Type,
Total_Revenue,
Avg_Revenue_Per_User
FROM (
SELECT
Country_table.Country AS Country,
Subscription_table.Subscription_Type AS Plan_Name,
SUM(NetflixUserbase.Monthly_Revenue *
(TIMESTAMPDIFF(MONTH, NetflixUserbase.Join_Date, NetflixUserbase.LMP)
+ 1)) AS Total_Revenue,
ROUND(AVG(NetflixUserbase.Monthly_Revenue *
(TIMESTAMPDIFF(MONTH, NetflixUserbase.Join_Date, NetflixUserbase.LMP)
+ 1)), 2) AS Avg_Revenue_Per_User
FROM NetflixUserbase
JOIN Country_table
ON NetflixUserbase.Country_Id = Country_table.Country_ID
JOIN Subscription_table
ON NetflixUserbase.Subscription_ID = Subscription_table.Subscription_code
GROUP BY Country_table.Country, Subscription_table.Subscription_Type WITH ROLLUP
) AS Revenue_Data
WHERE Country IS NOT NULL
ORDER BY Country, Plan_Name IS NULL, Total_Revenue DESC;
```

**Output:**

```
Country         Subscription_Type  Total_Revenue  Avg_Revenue_Per_User
Australia       Premium            13455          133.22
Australia       Standard           7187           140.92
Australia       Basic              4151           133.90
----------------------------------------------------------------------
Australia       Total              24793          135.48
----------------------------------------------------------------------
Brazil          Basic              19713          135.02
Brazil          Premium            4388           132.97
Brazil          Standard           654            163.50
----------------------------------------------------------------------
Brazil          Total              24755          135.27
----------------------------------------------------------------------
Canada          Basic              19822          136.70
Canada          Premium            11739          133.40
Canada          Standard           11092          132.05
----------------------------------------------------------------------
Canada          Total              42653          134.55
----------------------------------------------------------------------
France          Premium            20887          142.09
France          Basic              4926           136.83
----------------------------------------------------------------------
France          Total              25813          141.05
----------------------------------------------------------------------
Germany         Basic              19524          131.03
Germany         Standard           4126           133.10
Germany         Premium            394            131.33
----------------------------------------------------------------------
Germany         Total              24044          131.39
----------------------------------------------------------------------
Italy           Basic              22497          127.82
Italy           Premium            525            131.25
Italy           Standard           377            125.67
----------------------------------------------------------------------
Italy           Total              23399          127.86
----------------------------------------------------------------------
Mexico          Standard           23666          132.21
Mexico          Basic              384            96.00
----------------------------------------------------------------------
Mexico          Total              24050          131.42
----------------------------------------------------------------------
Spain           Premium            27804          131.15
Spain           Standard           16255          126.01
Spain           Basic              14693          133.57
----------------------------------------------------------------------
Spain           Total              58752          130.27
----------------------------------------------------------------------
United Kingdom  Standard           25389          141.05
United Kingdom  Basic              400            133.33
----------------------------------------------------------------------
United Kingdom  Total              25789          140.92
----------------------------------------------------------------------
United States   Basic              26507          133.20
United States   Premium            19045          131.34
United States   Standard           14376          134.36
----------------------------------------------------------------------
United States   Total              59928          132.88
----------------------------------------------------------------------
```

**Analysis:** The United States generates the most revenue (59,928) and Italy the least (23,399), with Spain close behind the United States at 58,752. France has the highest average revenue per user (141.05) but a total of only 25,813 because it has just 183 users, so focusing on France could generate more revenue even though its total is lower. Italy also has the lowest average revenue per user (127.86), so it has the most room to improve on both total and average.
```
### Question 5

**Question:** Which device does each gender prefer? Count the users of each gender on every device and show the total revenue they brought in, followed by the total of that gender.

**Code:**

```sql
SELECT
Gender,
IFNULL(Device_Name, 'Total') AS Device,
Users,
Total_Revenue
FROM (
SELECT
Gender_table.Gender AS Gender,
Device_table.Device AS Device_Name,
COUNT(*) AS Users,
-- Total revenue = Monthly_Revenue x months paid (from Join_Date to LMP, joining month counted)
SUM(NetflixUserbase.Monthly_Revenue *
(TIMESTAMPDIFF(MONTH, NetflixUserbase.Join_Date, NetflixUserbase.LMP)
+ 1)) AS Total_Revenue
FROM NetflixUserbase
JOIN Gender_table
ON NetflixUserbase.Gender_Id = Gender_table.Gender_Id
JOIN Device_table
ON NetflixUserbase.Device_Id = Device_table.Device_ID
GROUP BY Gender_table.Gender, Device_table.Device WITH ROLLUP
) AS Gender_Data
WHERE Gender IS NOT NULL
ORDER BY Gender, Device_Name IS NULL, Users DESC;
```

**Output:**

```
Gender  Device      Users  Total_Revenue
Female  Laptop      329    43485
Female  Tablet      323    44581
Female  Smart TV    305    39936
Female  Smartphone  300    39450
----------------------------------------
Female  Total       1257   167452
----------------------------------------
Male    Smartphone  321    43602
Male    Tablet      310    41346
Male    Laptop      307    40809
Male    Smart TV    305    40767
----------------------------------------
Male    Total       1243   166524
----------------------------------------
```

**Analysis:** Male and female users are almost the same, with females slightly ahead (1,257 users and 167,452 revenue against 1,243 users and 166,524 for males). Laptop is the most used device overall (329 female and 307 male users), though males lean towards smartphone, and the gap between devices is small. Since laptop users are the biggest group, a plan with better features at a slightly higher price could bring in more revenue from them.
```
### Question 6

**Question:** Which plans are chosen by users older than the average age? Find the average age of all users, then count the users above that age on each plan and show the total revenue they brought in.

**Code:**

```sql
SELECT
Subscription_table.Subscription_Type,
COUNT(*) AS Users,
-- Total revenue = Monthly_Revenue x months paid (from Join_Date to LMP, joining month counted)
SUM(NetflixUserbase.Monthly_Revenue *
(TIMESTAMPDIFF(MONTH, NetflixUserbase.Join_Date, NetflixUserbase.LMP)
+ 1)) AS Total_Revenue
FROM NetflixUserbase
JOIN Subscription_table
ON NetflixUserbase.Subscription_ID = Subscription_table.Subscription_code
WHERE NetflixUserbase.Age > (SELECT AVG(Age) FROM NetflixUserbase)
GROUP BY Subscription_table.Subscription_Type
ORDER BY Users DESC;
```

**Output:**

```
Subscription_Type  Users  Total_Revenue
Basic              519    68210
Standard           404    54737
Premium            366    49324
```
**Analysis:**
Users aged 39+ make up 52% of the user base and contribute 172,271 in revenue, making them a significant customer segment.
Their plan preference mirrors the overall user base, with Basic leading, followed by Standard and Premium.
```
```
4) Key Insights

Basic is the most popular and highest-revenue plan, while the U.S. and Spain are closely matched. France has the highest average revenue per user, whereas Italy performs lowest in both total and average revenue.

To increase revenue sustainably, focus on upgrading Basic users, improve Italy through targeted offers, and introduce value-added features or bundles for Laptop and Smart TV users.

5) use fo AI

   AI was used to **optimize and support the project**, mainly for refining SQL queries, simplifying code, and improving the README. The **analysis, questions, and logic were my own**, with AI helping me validate and present my work more effectively.


   
