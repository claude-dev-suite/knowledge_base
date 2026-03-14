# PL/pgSQL Deep Dive

## Architecture

PL/pgSQL is a loadable procedural language that:
- Compiles to bytecode on first execution
- Caches plans for SQL statements
- Runs inside PostgreSQL server process
- Has full access to database features

## Variable Declaration

```sql
DECLARE
    -- Basic types
    v_id INTEGER;
    v_name VARCHAR(100);
    v_amount NUMERIC(10,2) := 0.00;
    v_active BOOLEAN DEFAULT TRUE;

    -- Type from column
    v_user_email users.email%TYPE;

    -- Type from table row
    v_user users%ROWTYPE;

    -- Record (dynamic structure)
    v_record RECORD;

    -- Array
    v_ids INTEGER[] := ARRAY[1, 2, 3];
    v_names TEXT[] := '{"a", "b", "c"}';

    -- Composite type
    v_point point;

    -- Domain type
    v_email email_domain;

    -- Constant
    c_pi CONSTANT NUMERIC := 3.14159;

    -- NOT NULL (must have default)
    v_required INTEGER NOT NULL := 0;
```

## Control Structures

### Conditional

```sql
-- IF with multiple conditions
IF condition1 THEN
    -- statements
ELSIF condition2 THEN
    -- statements
ELSIF condition3 THEN
    -- statements
ELSE
    -- statements
END IF;

-- CASE expression
result := CASE status
    WHEN 'A' THEN 'Active'
    WHEN 'I' THEN 'Inactive'
    ELSE 'Unknown'
END;

-- Searched CASE
result := CASE
    WHEN age < 18 THEN 'Minor'
    WHEN age < 65 THEN 'Adult'
    ELSE 'Senior'
END;
```

### Loops

```sql
-- Infinite loop
LOOP
    -- Exit with condition
    EXIT WHEN counter > 100;
    -- Or CONTINUE to skip
    CONTINUE WHEN counter = 50;
    counter := counter + 1;
END LOOP;

-- While loop
WHILE counter < 100 LOOP
    counter := counter + 1;
END LOOP;

-- For loop (integer)
FOR i IN 1..10 LOOP
    RAISE NOTICE 'i = %', i;
END LOOP;

-- Reverse
FOR i IN REVERSE 10..1 LOOP
    RAISE NOTICE 'i = %', i;
END LOOP;

-- By step
FOR i IN 1..100 BY 10 LOOP
    RAISE NOTICE 'i = %', i;  -- 1, 11, 21, ...
END LOOP;

-- For loop (query)
FOR rec IN SELECT * FROM users LOOP
    RAISE NOTICE 'User: %', rec.name;
END LOOP;

-- For loop (dynamic query)
FOR rec IN EXECUTE 'SELECT * FROM ' || table_name LOOP
    -- process
END LOOP;

-- FOREACH (arrays)
FOREACH item IN ARRAY items LOOP
    RAISE NOTICE 'Item: %', item;
END LOOP;

-- FOREACH with slice
FOREACH item SLICE 1 IN ARRAY two_d_array LOOP
    -- item is a 1D array (row of 2D array)
END LOOP;
```

## Exception Handling

### Exception Categories

| Category | Example Errors |
|----------|----------------|
| Class 00 - Success | successful_completion |
| Class 02 - No Data | no_data_found |
| Class 22 - Data Exception | division_by_zero, string_data_truncation |
| Class 23 - Integrity Violation | unique_violation, foreign_key_violation |
| Class 40 - Transaction Rollback | deadlock_detected |
| Class 42 - Syntax/Access Error | undefined_table, insufficient_privilege |

### Exception Handling Patterns

```sql
BEGIN
    -- Attempt operation
    INSERT INTO users (email) VALUES (p_email);
EXCEPTION
    WHEN unique_violation THEN
        -- Handle duplicate
        RAISE NOTICE 'Email % already exists', p_email;
        RETURN NULL;

    WHEN not_null_violation THEN
        RAISE EXCEPTION 'Required field missing';

    WHEN check_violation THEN
        RAISE EXCEPTION 'Constraint violation: %', SQLERRM;

    WHEN OTHERS THEN
        -- Catch all
        RAISE EXCEPTION 'Unexpected error: % %', SQLSTATE, SQLERRM;
END;
```

### Exception Variables

```sql
EXCEPTION
    WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_sqlstate = RETURNED_SQLSTATE,
            v_message = MESSAGE_TEXT,
            v_detail = PG_EXCEPTION_DETAIL,
            v_hint = PG_EXCEPTION_HINT,
            v_context = PG_EXCEPTION_CONTEXT;

        RAISE NOTICE 'Error: % - %', v_sqlstate, v_message;
        RAISE NOTICE 'Detail: %', v_detail;
        RAISE NOTICE 'Context: %', v_context;
```

## Dynamic SQL

### EXECUTE

```sql
-- Simple execution
EXECUTE 'SELECT COUNT(*) FROM ' || quote_ident(table_name);

-- With INTO
EXECUTE 'SELECT COUNT(*) FROM ' || quote_ident(table_name)
INTO v_count;

-- With parameters (safe from injection)
EXECUTE 'SELECT * FROM users WHERE email = $1 AND status = $2'
INTO v_user
USING p_email, p_status;

-- Using format() for safe identifier quoting
EXECUTE format('SELECT * FROM %I WHERE %I = $1', table_name, column_name)
INTO v_result
USING p_value;
```

### format() Specifiers

| Specifier | Usage |
|-----------|-------|
| `%s` | String value |
| `%I` | Identifier (quoted if needed) |
| `%L` | Literal (safely quoted) |
| `%%` | Literal % |

```sql
EXECUTE format(
    'INSERT INTO %I (%I, %I) VALUES (%L, %L)',
    'users', 'name', 'email', v_name, v_email
);
```

## Performance Considerations

### Plan Caching

```sql
-- First execution: Plan created and cached
-- Subsequent: Cached plan reused

-- Force re-planning
EXECUTE 'SELECT * FROM users WHERE id = $1' USING v_id;
-- Dynamic SQL always re-plans
```

### STRICT Modifier

```sql
-- Raise error if not exactly one row
SELECT * INTO STRICT v_user FROM users WHERE id = p_id;
-- Raises NO_DATA_FOUND or TOO_MANY_ROWS

-- Without STRICT: sets v_user to NULL if no rows
SELECT * INTO v_user FROM users WHERE id = p_id;
```

## Debugging

```sql
-- RAISE levels
RAISE DEBUG 'Detailed debug info';    -- Usually not shown
RAISE LOG 'Log message';              -- Server log only
RAISE INFO 'Information';             -- Client + log
RAISE NOTICE 'Notice message';        -- Client + log
RAISE WARNING 'Warning message';      -- Client + log
RAISE EXCEPTION 'Error message';      -- Aborts transaction

-- With placeholders
RAISE NOTICE 'User % has % orders', v_user_id, v_count;

-- Custom error code
RAISE EXCEPTION 'Invalid input' USING ERRCODE = '22023';
```

## Cursors Deep Dive

### Bound vs Unbound

```sql
-- Bound cursor (query specified at declaration)
DECLARE
    bound_cur CURSOR FOR SELECT * FROM users;

-- Unbound cursor (query specified at open)
DECLARE
    unbound_cur REFCURSOR;
BEGIN
    OPEN unbound_cur FOR SELECT * FROM users;
END;
```

### Cursor Operations

```sql
OPEN cursor_name;
FETCH cursor_name INTO variables;
FETCH NEXT FROM cursor_name INTO variables;
FETCH PRIOR FROM cursor_name INTO variables;  -- Scrollable only
FETCH FIRST FROM cursor_name INTO variables;  -- Scrollable only
FETCH LAST FROM cursor_name INTO variables;   -- Scrollable only
FETCH ABSOLUTE n FROM cursor_name INTO variables;
FETCH RELATIVE n FROM cursor_name INTO variables;
MOVE cursor_name;  -- Move without fetching
CLOSE cursor_name;
```

### Scrollable Cursors

```sql
DECLARE
    scroll_cur SCROLL CURSOR FOR SELECT * FROM users;
BEGIN
    OPEN scroll_cur;
    FETCH LAST FROM scroll_cur INTO v_user;   -- Go to end
    FETCH PRIOR FROM scroll_cur INTO v_user;  -- Go back
    CLOSE scroll_cur;
END;
```
