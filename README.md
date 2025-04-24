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

What if we want to find some more in depth info amount debts? I am curious about our Dx codes and how they are paying. I can use this query to find out what Dx codes are paying well and vice-versa:

select   diagnosis_code, SUM(Allowed_Amount) - SUM(Paid_Amount) as Outstanding from dbo.claim_data
GROUP BY Diagnosis_code
ORDER BY Outstanding desc

![image](https://github.com/user-attachments/assets/bc718961-e222-4287-9e1e-fc2d604a7610)

As we can see Dx code A02.1 has the highest debt. My thought process now is why is that DX code being paid so little? I should research this further to see why and how we can fix this. 







