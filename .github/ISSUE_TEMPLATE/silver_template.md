---
name: Silver Template
about: If we want to add attributes to silver layer use this template
title: "[SILVER] "
labels: ''
assignees: ''
---

# Silver Layer Template

## **Acceptance Criteria**  
- Review STM with SD member and @tyturriff  
- Create a delta table in the `b_customer.staging_silver_rudderstack` schema based off the STM  
- Create a daily delta refresh stored procedure in the `b_customer.staging_silver_rudderstack` schema  
- Create a silver table in the `s_customer.rudderstack` schema based off the STM  
- Create a daily delta load stored procedure in the `s_customer.rudderstack` schema  
- Create a current materialized view in the `s_customer.rudderstack` schema based off of the historical table  
- Add DML and DDLs to GitHub SQL repo  
- Add a silver child step function module call to the `main.tf` file. Then add that child step function to the existing parent step function module call in GitHub  
- Execute a one-time historical load of the data from bronze to silver  
- Request @tyturriff to add appropriate permissions to new tables/views  
- Update Data Flow Diagram in Lucid  

---

## **Testing**  
- Test that distinct counts of `profile_ids` in bronze match the distinct count of `profile_ids` in the new silver table (excluding `%@mailinator.com` emails)  
  - mt.com Profile types are: `email_address`, `mtcom_anonymous_id`, `auth0_user_id`  
- Confirm that no `profile_id` values contain an `'@'` symbol  
- Confirm that no `profile_id` values for `profile_type = 'email_address'` contain a `'-'` symbol  
- Confirm that all required silver standard columns are never null  
- Seek formal sign-off with SD stakeholder before moving to close  

---

## **Additional Notes**  
- Pages (silver) table defined [here](#) for reference  

