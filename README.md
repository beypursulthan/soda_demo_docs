*Technical Documentation*

## 1. Introduction

This documentation explains how the automation works, step by step, including the logic behind each part of the workflow. The detailed reasoning behind decisions is recorded in the accompanying decision log.


## 2. Workflow Overview

This Make.com scenario processes incoming Google Form submissions, enriches the data using an AI model, evaluates lead quality using Python logic, checks for duplicates, assigns responsibilities, and finally sends a formatted Slack notification.

The workflow consists of ten main connections explained below.


## 3. Workflow Breakdown

### **Connection 1 — Webhook Trigger**

A Custom Webhook listens for new Google Forms submissions.
The form provides four fields:

* **Name**
* **Job Title**
* **Company Name**
* **Email Address**

This initiates the automation.


### **Connection 2 — Duplicate Company Check**

A search is performed in a Google Sheets database to determine whether the company name already exists.
This prevents creating redundant entries and helps identify returning customers.


### **Connection 3 — Router for Parallel Tasks**

A router splits the flow into parallel paths:

1. **Path A:** Add a new row to Google Sheets to maintain a running database of all submissions.
2. **Path B:** Send the submission data to the AI model for enrichment.

### **Connection 4 — AI Data Enrichment (Gemini 2.5 Flash)**

An API request sends the form data to the AI model.
The AI enriches and categorizes the data into structured enum-based attributes:

#### **1. Email Type — `is_generic_mail`**

Checks whether the email domain is generic (gmail, yahoo, etc.)

* `0` → company domain
* `1` → generic email

#### **2. Company Size — `company_size`**

AI estimates company size based on online data:

* `1` → < 100 employees
* `2` → 101–1000
* `4` → 1001–10,000
* `8` → > 10,000

#### **3. HQ Region — `company_hq_location_area`**

The company’s HQ is assigned to one of five regions:

* `us_canada`
* `northern_central_europe`
* `suthern_europe_middle_east_latin_america`
* `asia_pacific`
* `other`

#### **4. Important Industry Check — `in_important_industry`**

Marked as `1` if the company operates in:

* Banking
* Insurance
* Finance
  Otherwise → `0`.

#### **5. Seniority Level — `seniority_level`**

Estimated from the job title:

* `-1` → Intern
* `0` → Junior
* `1` → Senior
* `3` → Management
* `5` → VP
* `8` → C-Level

#### **6. Revenue — `revenue`**

AI estimates the company’s revenue in Euros.
Formatting uses German number notation (e.g., *12,5 Mio €*).
If unknown → `0`.

#### **7. HQ Location — `company_hq_location`**

Full HQ location in the format:
**"Country / City"** (German language)

---

### **Connection 5 — Lead Scoring (0CodeKit + Python)**

A Python script calculates a lead score based on AI attributes:

```python
score = {{12.result.company_size}} + {{12.result.seniority_level}}

if {{12.result.is_generic_mail}}:
    score = 0

if score < 10 and {{12.result.in_important_industry}} == 1:
    score = 5

if score > 10:
    result_string = "Burnt"
elif score > 5:
    result_string = "Hot"
elif score > 1:
    result_string = "Warm"
else:
    result_string = "Cold"

result = {'score': result_string}
```

The outcome is one of four categories:
**Burnt, Hot, Warm, Cold**


### **Connection 6 — New vs. Existing Customer Check**

Google Sheets is queried again to determine whether the company already exists in the database.

This information influences Slack message routing and labeling.


### **Connection 7 — Count Previous Contacts (0CodeKit + Python)**

A second Python script evaluates how many times the same company has contacted before:

```python
warning = ":warning: Customer contacted us {0} times before"

result = {'note': ""}

if {{35.`__IMTLENGTH__`}} > 0:
    result = {'note': warning.format({{35.`__IMTLENGTH__`}})}
```

If the customer has previous submissions, a warning note is added.

---

### **Connection 8–9 — Assignment Logic (Make.com Tools)**

#### **Region-based Sales Assignment**

Depending on `company_hq_location_area`, the lead is assigned to the correct sales representative:

* `us_canada` → US/Canada sales rep
* `northern_central_europe` → North/Central Europe rep
* `suthern_europe_middle_east_latin_america` → SE/ME/LatAm rep
* `asia_pacific` → APAC rep
* `other` → Sales Manager

#### **Lead Score Emoji**

Based on the final score string:

* Burnt
* Hot
* Warm
* Cold

An emoji is assigned to be displayed in Slack.

#### **Existing Customer Handling**

If the company already exists → reassign message to:

```
<!subteam^S09ST70E6P9|customer_support>
```

and label it:

**Customer contract is already signed. Please check with customer.**


### **Connection 10 — Final Slack Notification**

A formatted Slack message is sent using the combined results:

```
{{27.output}} {{34.output}}
Score: {{23.result.score}}
{{38.result.note}}

Company information:
  - Name: {{24.`Company name`}}
  - HQ: {{12.result.company_hq_location}}
  - Revenue: {{12.result.revenue}}

Customer Contact: <mailto:{{24.`Email adress`}}|{{24.`Email adress`}}>
```

The message includes:

* Sales assignment tag
* Score emoji
* Lead category
* Warning note (if any)
* Full company enrichment details
* Contact email properly formatted as a mailto link


## End.
