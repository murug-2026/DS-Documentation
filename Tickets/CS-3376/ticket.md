Key details
Description

We need to fix this across all reports using it.

```sql
LEFT JOIN "HonkAPI" hapi on (hapi.id= (
        SELECT honk.id 
        FROM public."HonkAPI" honk
        JOIN public."TypeHonkAPIRequestStatus" thas on thas.id = honk.status_id and thas.code <> 'REQUEST_ERROR'
        WHERE honk.case_id = cs.id
        ORDER BY honk.id desc 
        LIMIT 1
    )
)
```
example of the issue is https://compass.driven-solutions.com/view-case/306318/

was completed by TNS but did not appear in the billing report since it was later dispatched to HONK.
