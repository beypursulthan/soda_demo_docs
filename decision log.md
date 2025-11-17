# Decision Log

This document explains **why** each step of the automation was designed in its current form and the reasoning behind the key implementation choices.

## 1. Introduction

Task 2 was selected for this assignment because Task 1 closely overlaps with my previous professional experience and was already discussed extensively during the first-round interview. Task 2, on the other hand, allows working with something new and different. It involves AI-based automation, which is more creative and more aligned with current technological developments.

I chose Make.com for the implementation because it is convenient, simple to configure for a demo, offers 1000 credits for testing, and supports AI integrations, including Google Gemini, out of the box.


## 2. Webhook vs. Direct Google Forms Trigger

A **webhook** was used instead of the native Google Forms module.

* Make.com does not support real-time triggers for Google Forms.
* The built-in trigger only checks for new submissions every 15 minutes or at other intervals.
* For a **live demo**, waiting 15 minutes is not practical.
* A webhook ensures **instant processing** without requiring manual refreshes.

In a real-world scenario where real-time feedback is not necessary, the direct Google Forms connector would be sufficient (e.g., processing submissions at the end of the day). For the purposes of this assignment, real-time updates were needed, so webhook was the logical choice.


## 3. Creating a Database in Google Sheets

Although not explicitly required, I intentionally added a Google Sheets database for two reasons:

1. **Tracking Existing Customers**
   We must detect whether a company already exists in the database so the Slack message can correctly route the lead to **customer_support** instead of the **sales_rep**.

2. **Tracking Duplicate Submissions**
   Not all duplicate submissions indicate an existing customer. A company may submit the form multiple times before becoming a customer.

   * These entries should still go to **sales_rep**, but
   * They should include a note that this is not the first contact.

For both purposes, Sheets serves as a simple and effective data store.

This check must happen immediately after receiving the form data to avoid double-counting and to ensure accurate lead labeling.


## 4. Router for Parallel Processing

A router was added mainly for efficiency.

The AI enrichment step takes the longest. The Google Sheets update does not depend on the AI model, so both processes can run simultaneously.

* **Path A:** Add new row to Google Sheets
* **Path B:** Send data to the AI model

The workflow would work without parallelization, but this structure reduces overall processing time.


## 5. Choice of AI Model – Google Gemini 2.5 Flash

I chose the Gemini model primarily because:

* It offers a **free API tier** and is easily accessible from Google AI studio. It also offers access to all of its premium models for free to test.
* The integration with Make.com is straightforward.
* The initial use of **Gemini 2.5 Pro** was unnecessary for this task.
* After testing, I downgraded to **2.5 Flash**, which is significantly faster.
* The task does not require deep reasoning—only structured enrichment—so a lightweight model is more efficient.

This resulted in faster execution, lower usage costs, and fewer delays during testing.


## 6. Structured AI Output

The AI was configured to return **strictly structured data** using enum values. This was done for several reasons:

* Keeps the AI on-topic and minimizes hallucination risk.
* Makes downstream processing deterministic.
* Allows simple Python scripts to evaluate and calculate scores.
* Keeps the logic stable, even if the AI model evolves over time.

The enriched attributes include:

* `is_generic_mail`
* `company_size`
* `company_hq_location_area`
* `in_important_industry`
* `seniority_level`
* `revenue`
* `company_hq_location`

The revenue formatting uses German notation (`mio`, `tsd`). I chose German formatting simply because it aligned with the environment I was working in; this can be changed easily if needed.


## 7. Python Script for Lead Scoring

Make.com does not natively support Python, so I used **0CodeKit** to run the scoring logic externally.

The script uses the enriched values to calculate a score:

* The score starts as a combination of **company size** and **seniority level**.
* If the email is generic, the score is forced to zero.
* Important-industry leads with low initial scores are boosted to ensure they are not overlooked.
* The final score is categorized into **Cold**, **Warm**, **Hot**, or **Burnt**.

Although the original task required three levels, I chose four because:

* Sales teams often benefit from a more granular heat classification.
* “Burnt” is useful as an upper bound for leads that look too good to be true or have anomalies.
* It gives clearer differentiation and supports better prioritization.

AI could have generated the score without any Python logic, but I intentionally avoided this:

* AI models evolve, change behavior, and can be occasionally unreliable.
* The more responsibilities the AI has, the more time it takes and the greater the risk of inconsistent output.
* Keeping the AI’s role minimal reduces complexity and avoids hidden model-dependent logic.

Python offers deterministic, transparent scoring.
  

## 8. Second Customer Check – Counting Repeat Form Submissions

A second Google Sheets lookup determines whether the company has submitted multiple times **within the form system** (not the CRM).

This is different from checking whether they are already a customer.

For example:

* A company submits the form twice → Not a customer yet → Should still go to **sales_rep**, but with a “repeat contact” notice.

A small Python snippet (via 0CodeKit again) adds a warning message whenever the count is greater than zero.

This ensures the sales rep immediately knows if this is a follow-up inquiry.


## 9. Final Routing Using Make.com Tools

The three built-in Make.com tools handle:

1. **Sales rep assignment based on the HQ region**
2. **Emoji assignment based on score**
3. **Sales_rep vs. customer_support routing based on database checks**

These tools execute simple conditional logic and formatting to prepare the final Slack message.

I deliberately included emojis.
In this context, they are functional:

* They make the score highly visible.
* They help sales reps instantly understand the lead temperature.
* The warning symbol highlights repeat contacts.

Here, the use of emojis improves usability.


End of Decision Log.
