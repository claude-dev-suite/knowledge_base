# Oracle PL/SQL Deep Dive

## PL/SQL Architecture

PL/SQL engine:
- Parses and compiles PL/SQL code
- Executes procedural statements
- Sends SQL to SQL engine
- Minimizes context switches between engines

## Block Structure

```sql
<<block_label>>
DECLARE
    -- Declarations (optional)
BEGIN
    -- Executable statements (required)
EXCEPTION
    -- Exception handlers (optional)
END block_label;
/
```

## Data Types

### Scalar Types

```sql
DECLARE
    -- Numeric
    v_integer INTEGER;                    -- -2^31 to 2^31
    v_pls_integer PLS_INTEGER;           -- Faster integer (-2^31 to 2^31)
    v_binary_integer BINARY_INTEGER;     -- Same as PLS_INTEGER
    v_number NUMBER(10,2);               -- Fixed precision
    v_binary_float BINARY_FLOAT;         -- IEEE 754 single
    v_binary_double BINARY_DOUBLE;       -- IEEE 754 double

    -- Character
    v_char CHAR(10);                     -- Fixed length
    v_varchar2 VARCHAR2(4000);           -- Variable length
    v_long LONG;                         -- Up to 2GB (deprecated)
    v_clob CLOB;                         -- Up to 4GB
    v_nchar NCHAR(10);                   -- Unicode fixed
    v_nvarchar2 NVARCHAR2(4000);        -- Unicode variable
    v_nclob NCLOB;                       -- Unicode large

    -- Date/Time
    v_date DATE;
    v_timestamp TIMESTAMP(6);
    v_timestamp_tz TIMESTAMP WITH TIME ZONE;
    v_interval_ym INTERVAL YEAR TO MONTH;
    v_interval_ds INTERVAL DAY TO SECOND;

    -- Boolean
    v_boolean BOOLEAN;  -- TRUE, FALSE, NULL

    -- Binary
    v_raw RAW(2000);
    v_blob BLOB;
```

### PLS_INTEGER vs NUMBER

```sql
-- PLS_INTEGER is faster for integer arithmetic
-- Uses native machine arithmetic
-- Raises exception on overflow (not silent wrap)

DECLARE
    v_pls PLS_INTEGER := 0;
    v_num NUMBER := 0;
BEGIN
    FOR i IN 1..1000000 LOOP
        v_pls := v_pls + 1;  -- Faster
        v_num := v_num + 1;  -- Slower
    END LOOP;
END;
```

## Subtypes

```sql
DECLARE
    -- Built-in subtypes
    v_natural NATURAL;           -- PLS_INTEGER >= 0
    v_naturaln NATURALN := 0;    -- NATURAL NOT NULL
    v_positive POSITIVE;         -- PLS_INTEGER > 0
    v_positiven POSITIVEN := 1;  -- POSITIVE NOT NULL
    v_signtype SIGNTYPE;         -- -1, 0, or 1
    v_simple_integer SIMPLE_INTEGER := 0;  -- NOT NULL, no overflow check

    -- Custom subtype
    SUBTYPE currency IS NUMBER(10,2);
    v_price currency;
```

## %TYPE and %ROWTYPE

```sql
DECLARE
    -- Match column type
    v_email employees.email%TYPE;

    -- Match entire row
    v_employee employees%ROWTYPE;

    -- Match cursor row
    CURSOR emp_cur IS SELECT employee_id, first_name FROM employees;
    v_emp emp_cur%ROWTYPE;

    -- Match record type
    TYPE emp_rec IS RECORD (
        id NUMBER,
        name VARCHAR2(100)
    );
    v_rec emp_rec;
    v_rec2 v_rec%TYPE;  -- Same type as v_rec
```

## Collections

### Associative Array (INDEX BY)

```sql
DECLARE
    TYPE string_array IS TABLE OF VARCHAR2(100) INDEX BY PLS_INTEGER;
    TYPE emp_array IS TABLE OF employees%ROWTYPE INDEX BY VARCHAR2(20);

    v_names string_array;
    v_emps emp_array;
BEGIN
    v_names(1) := 'John';
    v_names(2) := 'Jane';

    v_emps('EMP001') := emp_rec;

    -- Iterate
    FOR i IN v_names.FIRST..v_names.LAST LOOP
        DBMS_OUTPUT.PUT_LINE(v_names(i));
    END LOOP;
END;
```

### Nested Table

```sql
DECLARE
    TYPE number_tab IS TABLE OF NUMBER;
    v_numbers number_tab := number_tab(10, 20, 30);
BEGIN
    v_numbers.EXTEND;           -- Add element
    v_numbers(4) := 40;

    v_numbers.DELETE(2);        -- Remove element (creates gap)

    FOR i IN 1..v_numbers.COUNT LOOP
        IF v_numbers.EXISTS(i) THEN
            DBMS_OUTPUT.PUT_LINE(v_numbers(i));
        END IF;
    END LOOP;
END;
```

### VARRAY

```sql
DECLARE
    TYPE phone_varray IS VARRAY(5) OF VARCHAR2(20);
    v_phones phone_varray := phone_varray('555-1234', '555-5678');
BEGIN
    -- Cannot exceed declared size (5)
    v_phones.EXTEND;
    v_phones(3) := '555-9999';
END;
```

### Collection Methods

| Method | Description |
|--------|-------------|
| COUNT | Number of elements |
| EXISTS(n) | TRUE if element n exists |
| FIRST | First index |
| LAST | Last index |
| NEXT(n) | Index after n |
| PRIOR(n) | Index before n |
| EXTEND | Add element(s) |
| TRIM | Remove from end |
| DELETE | Remove element(s) |
| LIMIT | Maximum size (VARRAY only) |

## Bulk Operations

### BULK COLLECT

```sql
DECLARE
    TYPE emp_tab IS TABLE OF employees%ROWTYPE;
    v_employees emp_tab;
BEGIN
    -- Fetch all at once
    SELECT * BULK COLLECT INTO v_employees
    FROM employees WHERE department_id = 10;

    -- With LIMIT
    DECLARE
        CURSOR emp_cur IS SELECT * FROM employees;
    BEGIN
        OPEN emp_cur;
        LOOP
            FETCH emp_cur BULK COLLECT INTO v_employees LIMIT 1000;
            EXIT WHEN v_employees.COUNT = 0;
            -- Process batch
        END LOOP;
        CLOSE emp_cur;
    END;
END;
```

### FORALL

```sql
DECLARE
    TYPE id_tab IS TABLE OF NUMBER;
    v_ids id_tab := id_tab(100, 101, 102);
BEGIN
    FORALL i IN 1..v_ids.COUNT
        UPDATE employees SET salary = salary * 1.1
        WHERE employee_id = v_ids(i);

    -- With SAVE EXCEPTIONS
    FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
        INSERT INTO audit_log VALUES (v_ids(i), SYSDATE);

    EXCEPTION
        WHEN OTHERS THEN
            FOR j IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
                DBMS_OUTPUT.PUT_LINE('Error at index ' ||
                    SQL%BULK_EXCEPTIONS(j).ERROR_INDEX || ': ' ||
                    SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE));
            END LOOP;
END;
```

## RETURNING Clause

```sql
DECLARE
    v_id NUMBER;
    v_ids number_tab;
BEGIN
    -- Single row
    INSERT INTO employees (first_name, salary)
    VALUES ('John', 50000)
    RETURNING employee_id INTO v_id;

    -- Multiple rows with BULK COLLECT
    UPDATE employees SET salary = salary * 1.1
    WHERE department_id = 10
    RETURNING employee_id BULK COLLECT INTO v_ids;
END;
```

## Named Notation

```sql
CREATE PROCEDURE process_order(
    p_order_id IN NUMBER,
    p_status IN VARCHAR2 DEFAULT 'PENDING',
    p_priority IN NUMBER DEFAULT 1
) IS BEGIN NULL; END;
/

-- Call with named notation
BEGIN
    process_order(p_order_id => 123, p_priority => 5);
    -- p_status uses default
END;
```

## PRAGMA Directives

```sql
DECLARE
    -- Associate exception with error code
    e_no_data EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_no_data, -20001);

    -- Restrict side effects (for SQL functions)
    -- WNDS: Writes No Database State
    -- RNDS: Reads No Database State
    -- WNPS: Writes No Package State
    -- RNPS: Reads No Package State
    PRAGMA RESTRICT_REFERENCES(my_function, WNDS, RNDS);

    -- Mark as autonomous transaction
    PRAGMA AUTONOMOUS_TRANSACTION;

    -- Inline for performance
    PRAGMA INLINE(my_procedure, 'YES');
```

## NOCOPY Hint

```sql
-- Pass by reference instead of value (performance for large parameters)
PROCEDURE process_large_data(
    p_data IN OUT NOCOPY large_collection_type
) IS
BEGIN
    -- Changes visible immediately
    -- Rollback semantics different (no copy to restore)
END;
```
