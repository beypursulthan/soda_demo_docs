
# Brainstorming

## Score Calculation

- Input
    - !Generic Mail
    - Company Size
        - < 100 => 1
        - 101-1000 => 2
        - 1001-10000 => 4
        - \> 10000 => 8
    - Seniority of Role
        - Intern => -1
        - Junior => 0
        - Senior => 1
        - Management => 3
        - VP => 5
        - C-Level => 8
    - Industry
        - !banking/insurance/financial 
- Output
    - Cold <= 1
    - Warm > 1
    - Hot > 5
    - Burnt > 10
    

## Flags

- If existing customer => Tag customer_support instead of sales_rep
- If not => Tag Sales_rep based on location
    - If location is "other" tag sales_manager
- Add mail_to link for email
- Include Revenue of company
- Flag if they contacted us before (based on sheet)

## Next steps

- If too many messages per day
    - Group into locations & score => 3 groups per sales_rep
    - Send accumulated messages at the end of the day instead of as soon as formular is submitted
    - Less than 50 => 2 messages per day; If more => 1 message per day
    - If receiving multiple leads by same company per day => Merge them
- Connect CRM system & include communication history with customer => Use gemini/AI to summarize the communicaiton hitory (if complying with data privacy)
- If e.g. Amazon India and Amazon US are both contacting us tag sales_manager to further coordinate

###