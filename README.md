# HealthData
SQL + Tableau for Analyzing healthcare claim data
This is a testing ground for SQL and Tableau data Analysis. Data used is from https://www.kaggle.com/datasets/abuthahir1998/synthetic-healthcare-claims-dataset. 

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















