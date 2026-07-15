---
tags:
  - database
  - postgresql
---

Databases, particularly those supporting three-valued logic (3VL), incorporate TRUE, FALSE, and UNKNOWN to handle uncertainty and missing information in a robust way. This design is rooted in the principles of relational databases and SQL, where handling incomplete or uncertain data is common. Here's a concise explanation of why this design exists and its reasoning:

- **Handling NULL Values**:
    - In databases, **NULL** represents missing, undefined, or inapplicable data. For example, if a column like birth_date is NULL for a record, it means the birth date is unknown.
    - When evaluating conditions involving NULL (e.g., age > 30 where age is NULL), the result isn't definitively TRUE or FALSE because the value is unknown. This leads to the third state: **UNKNOWN**.
- **Reflecting Real-World Uncertainty**:
    - Real-world data often involves uncertainty or incomplete information. For instance, in a medical database, a patient's test result might be pending, so a condition like test_result = 'positive' can't be evaluated as TRUE or FALSE yet—it’s UNKNOWN.
    - 3VL ensures the database accurately represents this uncertainty rather than forcing a binary (TRUE/FALSE) outcome, which could lead to incorrect assumptions
    

- **Logical Consistency in SQL**: 
- SQL uses 3VL to maintain consistent behavior when evaluating logical expressions. For example:
    - TRUE AND UNKNOWN = UNKNOWN (because the outcome depends on the unknown value).
    - TRUE OR UNKNOWN = TRUE (because TRUE is sufficient for OR to be TRUE).
    - FALSE AND UNKNOWN = FALSE (because FALSE makes the AND FALSE regardless of the other value).
- This ensures predictable and mathematically sound behavior when combining conditions with NULLs.