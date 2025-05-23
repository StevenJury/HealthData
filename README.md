-------------FURTHER READING FOUND IN WIKI-------------

# HealthData
SQL + Tableau for Analyzing healthcare claim data
This is a testing ground for SQL and Tableau data Analysis. Data used is from https://www.kaggle.com/datasets/abuthahir1998/synthetic-healthcare-claims-dataset.  
https://www.cms.gov/medicare/coordination-benefits-recovery/overview/icd-code-lists
https://www.kaggle.com/datasets/kalyankumarm/dataset

CSV imported onto Tableau and SQL. I am using Microsoft SQL Server Management(SSMS). Table setup is as follows :

Claim_ID	nvarchar(50)	
Provider_ID	bigint	
Patient_ID	bigint	
Date_of_Service	date	
Billed_Amount	smallint	
Procedure_Code	int	Checked
Diagnosis_Code	nvarchar(50)	
Allowed_Amount	smallint	
Paid_Amount	smallint	
Insurance_Type	nvarchar(50)	
Claim_Status	nvarchar(50)	
Reason_Code	nvarchar(50)	
Follow_up_Required	bit	
AR_Status	nvarchar(50)	
Outcome	nvarchar(50)	

My first step is determining the outstanding balance of our claims. This helps us determine certain payor types or dx codes that are showing a significantly high amount of outstanding balances. This can be accomplished quite easily. All I need to do is get the SUM of my Allowed Amount, which shows how much we were expected to be paid, and subtract is by the SUM of the Paid Amount. For example a allowed amount of 1000 and a paid amount of 900 leaves us a balance of $100. This number is helpful but by using the GROUP BY function I can find out some more important information. For instance I was curious about how much of a balance is left for each of our payor types. This information can be used to determine which type to focus your AR efforts on. For example, if our balance is highest in Medicare, we might want to focus more efforts towards resolving Medicare AR debt or try and find out WHY the balance is so high for that type. We can find out this information with a simple query:


select insurance_type, SUM(Allowed_Amount) - SUM(Paid_Amount) as Outstanding from dbo.claim_data
GROUP BY Insurance_type
ORDER BY Outstanding asc

![image](https://github.com/user-attachments/assets/3b150c9a-2a39-4446-94c0-4be58331ef50)


I am choosing to order this data by the lowest debt to highest, but that is just a personal choice.  This table shows us that Self-Pay has the highest debt, which is not suprising. After this I can enter the same data on Tableau to see if it matches, For example I would create a pill for SUM(Allowed Amount) - SUM(Paid_Amount), move Insurance type to my column, and then select Text Table, Review of this shows that the data matches. I then can move my new Aggregate Function over to text and color, and choose red-green diverging reversed to show a deeper red for higher debt.


![image](https://github.com/user-attachments/assets/1821a0b6-4c67-4d51-ba6d-c2a87d2a19db)

What if we want to find some more in depth info about  debts? I am curious about our Dx codes and how they are paying. I can use this query to find out what Dx codes are paying well and vice-versa:

select   diagnosis_code, SUM(Allowed_Amount) - SUM(Paid_Amount) as Outstanding from dbo.claim_data
GROUP BY Diagnosis_code
ORDER BY Outstanding desc

![image](https://github.com/user-attachments/assets/bc718961-e222-4287-9e1e-fc2d604a7610)

As we can see Dx code A02.1 has the highest debt. My thought process now is why is that DX code being paid so little? I should research this further to see why and how we can fix this.
To see this on Tableau We can create an Agg Function again for Debt and name it 'Debt". and put in on the Rows and then put Dx code on the Column. This creates a bar graph showing all of the Dx codes and their respective debt. I decided to filter to debts higher than $300 and use the same color format as earlier. Once again our debt numbers match with our SQL Queries. This same exact process can also be done with procedure codes. 

![image](https://github.com/user-attachments/assets/2511d3e1-735d-4201-acd3-faf66d04766c)

Or reason codes: 

![image](https://github.com/user-attachments/assets/677a7a25-9276-46a0-b2f2-9379eae60edc)


For my next query I want to use division in an aggregate function but I am only returning 0's and 1's. This is due to my CSV being uploaded as a flat file with default column types. SSMS made my Billed, Allowed, and Paid amount integers, which means they will only return another int. To fix this these columns must be reclassified as floats and the table recreated. New table:dbo.claim_datanew

We can use percentages to get another look at the data, for example, I want to see what percentage of a claim paid. I can accomplish this by dividing my paid amount by my allowed amount and multiplying that by 100. I am using the ROUND function to make the data more presentable, and using CONCAT to add a percentage. The function must also use SUM in order to be grouped properly. I decided to use this to see how our procedure codes were being paid: 

select  procedure_code, CONCAT(ROUND((SUM(paid_amount) / SUM(allowed_amount) * 100),2),'%') as percent_paid from dbo.claim_datanew
GROUP BY procedure_code
ORDER BY percent_paid asc


![image](https://github.com/user-attachments/assets/e452ddfe-dba1-4cd2-9dd1-2de491f901c8)

Now I am curious on this percentage and how it changes throughout the months. I can accomplish this by using the same SUM function as earlier for debt and using MONTH to extract the month from our Date_Of_Service Column. I then group by Month to see the percentage that claims are paid throughout the Months. I upped the decimal amount on this example just so it was a little more detailed

select DISTINCT MONTH(Date_of_Service) as month,  CONCAT(ROUND((SUM(paid_amount) / SUM(allowed_amount) * 100),5),'%') as percent_paid from dbo.claim_datanew
GROUP BY MONTH(Date_of_Service)
ORDER BY MONTH(Date_Of_Service) Desc

![image](https://github.com/user-attachments/assets/4e35aae5-0b54-4943-96d0-6a7d73405178)

As you can see this query wasnt extremely helpful, but if our dataset was larger we might gain some valuable insights. 

What if I want to find out what our most used Diagnosis Codes are? This can be accomplished with a simple query. We just need to use the COUNT function to see how many Dx codes there are (Making sure to NOT use DISTINCT as we want to see how many of each dx exist). Then we can Group by our Dx codes and order by our COUNT to see how many of each Dx exist. I decided to use the TOP function to see only the top 10, this function usually goes by the syntax LIMIT. 

![image](https://github.com/user-attachments/assets/17fa2f52-e280-4c8c-bd0e-97abc114252c)



I want to get more specific and see what claims have an unusually high debt. I can accomplish this by Getting the percentage that a claim was paid by using our CONCAT from earlier but removing SUM. I can then simply use a WHERE close to remove any 100% paid and order by our CONCAT.

select TOP 5 claim_id, patient_id, date_of_service, CONCAT(ROUND(((paid_amount) / (allowed_amount) * 100),5),'%') as paidpercent from dbo.claim_datanew
WHERE CONCAT(ROUND(((paid_amount) / (allowed_amount) * 100),5),'%') != '100%'
ORDER BY CONCAT(ROUND(((paid_amount) / (allowed_amount) * 100),5),'%') asc


![image](https://github.com/user-attachments/assets/bcb94371-2c6a-4a32-ae1d-3087ee9f9e45)


Upping our LIMIT/TOP to 50 shows that none of these are really outliers, but it is still interesting 

![image](https://github.com/user-attachments/assets/985d4d00-17b9-4e9b-906a-55b377def23e)


This data comes from multiple providers. Lets find out what providers are being paid and if anything looks weird. We can do this by simply updating our previous query 

select   DISTINCT TOP 10  provider_id, CONCAT(ROUND((SUM(paid_amount) / SUM(allowed_amount) * 100),2),'%') as percent_paid from dbo.claim_datanew
WHERE CONCAT(ROUND(((paid_amount) / (allowed_amount) * 100),5),'%') != '100%'
GROUP BY provider_id
ORDER BY  CONCAT(ROUND((SUM(paid_amount) / SUM(allowed_amount) * 100),2),'%') asc

![image](https://github.com/user-attachments/assets/33a8cf80-f1f6-4aac-9242-c49a7d9b3f4a)

In Tableau 

![image](https://github.com/user-attachments/assets/157138df-a148-4893-8120-08d75379017e)


What if I want to see how many times a Diagnosis/Procedure code combination shows up? This could be useful if we want to see how many similar claims we are sending out. 
We can use the COUNT function and DISTINCT to find out: 

select   DISTINCT Diagnosis_Code, Procedure_Code, Count(*) as Total from dbo.claim_datanew
GROUP BY diagnosis_code, procedure_code
ORDER BY total desc


![image](https://github.com/user-attachments/assets/00865811-1fef-4f3f-9928-5fd44f6a272f)



Now I want to see the amount of debt per combination. Lets add a SUM function :
select   DISTINCT Diagnosis_Code, Procedure_Code, Count(*) as total, SUM(allowed_amount - paid_amount) AS total_debt  from dbo.claim_datanew
GROUP BY diagnosis_code, procedure_code
ORDER BY total_debt desc

![image](https://github.com/user-attachments/assets/a7482ad3-7050-469b-8b8d-0f2437b87268)

As you can see, nothing super interesting stands out. 

What if I want to find out what the Diagnosis Codes actually mean? I can connect this database with a  ICD10 spreadsheet to find out the descriptions. I will be using a LEFT JOIN for this as I want to see what NULL values show up. This is important later. 
Our new ICD spreadsheet, obtained through https://www.cms.gov/medicare/coordination-benefits-recovery/overview/icd-code-lists, shows Diagnosis Codes without the Period string. For example A00.1 is A001. This can cause issues when using a JOIN, since we want to match the exact data between the two tables. To solve this I am going update our Diagnosis codes on our original table in order to remove the period. This can be accomplished with a simple UPDATE query: 

update dbo.claim_data
set Diagnosis_Code = replace(Diagnosis_Code, '.', '')

Now that our data matches, I will proceed with a left join using our Diagnosis Code as the joining column: 


select Diagnosis_Code, Description from dbo.claim_data c
LEFT JOIN dbo.ICD10  I ON c.Diagnosis_Code = I.Dxcode

![image](https://github.com/user-attachments/assets/97d59df6-01e1-4ed9-8db5-3f55dbf30901)



Upon review I am getting some NULL values, that is weird, the ICD spreadsheet I have linked to should have every single valid ICD10 code for 2024. This is because my simulated claims spreadsheet is using some nonexistant ICD10 codes as well as some simplified codes. 
When reviewing our claims that pulled descriptions, it looks like they all pulled the correct description from our new table. This can make viewing the overall data much easier


![image](https://github.com/user-attachments/assets/87381370-bd6d-47c8-9501-801152428ba4)


![image](https://github.com/user-attachments/assets/cf88c3ea-733f-4ce7-b51f-3938a986aded)

We can use this new join with one of our previous queries to see our debt per Dx code and a description so we know what that code actually means. This can also be applied to Procedure codes. 


select Diagnosis_Code, SUM(Billed_Amount) - SUM(Paid_Amount) as Debt, Description from dbo.claim_data c
LEFT JOIN dbo.ICD10  I ON c.Diagnosis_Code = I.Dxcode
WHERE Description IS NOT NULL
GROUP BY Diagnosis_Code, Description
ORDER BY DEBT desc

![image](https://github.com/user-attachments/assets/81361425-d8a4-4b66-859d-765a394b8de3)

What if we want to find how many times a certain decription/dx code matches with a procedure code? We can simple count our procedure codes and group by description then procedure code. I am also making sure I put in a WHERE clause due to those NULLS we encountered earlier. 

select COUNT(procedure_code),  Description from dbo.claim_data c
LEFT JOIN dbo.ICD10 I ON c.Diagnosis_Code = I.Dxcode 
WHERE DESCRIPTION IS NOT NULL
GROUP BY Description, procedure_code
ORDER BY COUNT(procedure_code) desc


![image](https://github.com/user-attachments/assets/94b37060-1b4c-4f13-b884-027740ece72a)


I am going to move onto a new dataset now. This project starts with patient demographic and charges. Database obtained from https://www.kaggle.com/datasets/kalyankumarm/dataset.
Uploaded onto SSMS with flat file type. 


![image](https://github.com/user-attachments/assets/47df5c7f-0c9b-43fd-b58e-4a393f0f197c)


Using select * to see what our data looks like: 

![image](https://github.com/user-attachments/assets/a4da576d-1ec8-4701-bc94-4fba7066b1a1)



First I am curious about charges by gender, I just want to see what gender is incurring more charges as this might be helpful data later on. 

SELECT SUM(Charges) as totalcharges, sex FROM dbo.claimsdata2
GROUP BY sex

As we can see from this males have more charges than female. 

![image](https://github.com/user-attachments/assets/a820fd5e-c806-4f3c-890e-87c53fef5f77)


Now I am curious about how our charges relate to the sex and age of the patient. This can be useful for predicting future charges for at risk demographics. I want to count the SUM of my charges and then group by age and sex like this: 

SELECT DISTINCT SUM(Charges) as totalcharges, sex,  age FROM dbo.claimsdata2
GROUP BY age, sex
ORDER BY totalcharges desc

![image](https://github.com/user-attachments/assets/0c9a8069-3580-4d1e-a83e-32c56b0910fa)



As we can see, males aged 19 are seeing the highest charges, interesting. 

Now I want to see how our charges look depending on how many kids a person has. To do this I am going to do a CASE statement. I could edit this to be more detailed if I wanted charges for each amount of kids, but I will just do two categories for now: 

SELECT 
	CASE 
		WHEN children >= 3 THEN '3 or More' 
		WHEN children < 3 THEN 'Less than 3' 
END AS Charges_PER_KID,
		SUM(Charges) as totalcharges
FROM dbo.claimsdata2
GROUP BY 
    CASE 
        WHEN children >= 3 THEN '3 or More'
        WHEN children < 3 THEN 'Less than 3'
    END;


![image](https://github.com/user-attachments/assets/923b6be0-77d9-4f9d-ad89-4a1413ebf3b8)

As we can see 3 or more kids is showing more charges. 

I am curious about how many people we have per region as we might be able to use this data later 

select SUM(charges) as total_charges, count(*) as total_people, region from dbo.claimsdata2
GROUP BY region

![image](https://github.com/user-attachments/assets/2c415b54-0bb2-4a30-ab97-3750c65f574f)



Now lets see charges per age group. This can help us determine which age group is being charges the most on average:
SELECT 
    CASE 
        WHEN age BETWEEN 18 AND 29 THEN '18-29'
        WHEN age BETWEEN 30 AND 39 THEN '30-39'
        WHEN age BETWEEN 40 AND 49 THEN '40-49'
        WHEN age BETWEEN 50 AND 59 THEN '50-59'
        WHEN age >= 60 THEN '60+'
    END AS age_group,
    AVG(charges) AS avg_charges
FROM 
    dbo.claimsdata2
GROUP BY 
    CASE 
        WHEN age BETWEEN 18 AND 29 THEN '18-29'
        WHEN age BETWEEN 30 AND 39 THEN '30-39'
        WHEN age BETWEEN 40 AND 49 THEN '40-49'
        WHEN age BETWEEN 50 AND 59 THEN '50-59'
        WHEN age >= 60 THEN '60+'
    END
ORDER BY age_group;

As expected, 60+ age bucket is being charged the most. 

![image](https://github.com/user-attachments/assets/d1a3b55b-1fcd-4cea-b6d1-a3f8dcd10165)



Now Im curious about the top 5 highest charged people per region, lets use a window function in order to rank people within their regions: 

SELECT 
    age,
    sex,
    bmi,
    children,
    smoker,
    region,
    charges
FROM (
    SELECT 
        age,
        sex,
        bmi,
        children,
        smoker,
        region,
        charges,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY charges DESC) AS rank_within_region
    FROM dbo.claimsdata2
) AS ranked
WHERE rank_within_region <= 5
ORDER BY region, rank_within_region;

![image](https://github.com/user-attachments/assets/40216ff0-2a29-48d4-83ac-3a6799202d0b)


This healthcare data can be crucial for getting a better in depth look at how claim processes can be improved. All of these SQL querys can be entered in Tableau to gain even further insights. 

