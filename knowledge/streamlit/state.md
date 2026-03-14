# Streamlit - State Management

> Official Documentation: https://docs.streamlit.io/develop/concepts/architecture/session-state

## Overview

Streamlit reruns the entire script on every user interaction, which means local variables are reset each time. State management covers three mechanisms to persist and share data: **Session State** (per-user, per-session variable persistence), **Caching** (skip re-executing expensive functions), and **Fragments** (rerun only part of the script).

---

## Table of Contents

1. [Session State Basics](#session-state-basics)
2. [Accessing & Updating Session State](#accessing--updating-session-state)
3. [Widget Keys & Session State](#widget-keys--session-state)
4. [Callbacks](#callbacks)
5. [Forms](#forms)
6. [st.cache_data](#stcache_data)
7. [st.cache_resource](#stcache_resource)
8. [Cache Parameters Reference](#cache-parameters-reference)
9. [Fragments](#fragments)
10. [Patterns & Best Practices](#patterns--best-practices)

---

## Session State Basics

`st.session_state` is a dictionary-like object that persists across reruns **within a single browser session**. Each user/tab gets its own isolated session state.

```python
import streamlit as st

# Initialize (always check before setting to avoid overwriting on rerun)
if "count" not in st.session_state:
    st.session_state.count = 0

st.write(f"Count: {st.session_state.count}")
```

Session State is lost when:
- The user closes the tab
- The Streamlit server restarts
- The user navigates away (unless using `st.navigation` / `st.Page`)

---

## Accessing & Updating Session State

Both dictionary-style and attribute-style access are supported:

```python
# Dictionary style
st.session_state["key"] = "value"
value = st.session_state["key"]

# Attribute style
st.session_state.key = "value"
value = st.session_state.key

# Check existence
if "key" in st.session_state:
    del st.session_state["key"]  # Delete a key

# Iterate
for key in st.session_state:
    st.write(key, st.session_state[key])
```

Accessing a key that doesn't exist raises a `KeyError`. Always initialize with a guard:

```python
if "data" not in st.session_state:
    st.session_state.data = []
```

### Counter Example

```python
import streamlit as st

if "count" not in st.session_state:
    st.session_state.count = 0

col1, col2, col3 = st.columns(3)
if col1.button("Increment"):
    st.session_state.count += 1
if col2.button("Decrement"):
    st.session_state.count -= 1
if col3.button("Reset"):
    st.session_state.count = 0

st.metric("Count", st.session_state.count)
```

### Storing Complex Objects

```python
import pandas as pd

if "history" not in st.session_state:
    st.session_state.history = []

new_item = st.text_input("Add item")
if st.button("Add") and new_item:
    st.session_state.history.append(new_item)

st.write("History:", st.session_state.history)

# Store DataFrames, models, etc.
if "df" not in st.session_state:
    st.session_state.df = pd.DataFrame()
```

---

## Widget Keys & Session State

Every widget can be assigned a `key`. When a key is set, the widget's current value is automatically stored in and synced with `st.session_state`:

```python
import streamlit as st

# Widget value auto-stored in session_state["my_slider"]
st.slider("Temperature", -100.0, 100.0, 20.0, key="my_slider")

st.write("Current temperature:", st.session_state.my_slider)
```

You can also pre-initialize a widget's value through session state:

```python
if "celsius" not in st.session_state:
    st.session_state.celsius = 20.0

st.slider("Celsius", -100.0, 100.0, key="celsius")
```

**Limitations:**
- Cannot set values for `st.button` or `st.file_uploader` via session state
- Widget keys must be unique across the app

---

## Callbacks

Callbacks are functions executed **before** the rest of the script runs, triggered by widget interactions. Use `on_change` (for input widgets) or `on_click` (for buttons).

### Basic Callback

```python
import streamlit as st

if "count" not in st.session_state:
    st.session_state.count = 0

def increment():
    st.session_state.count += 1

def decrement():
    st.session_state.count -= 1

st.button("+ Increment", on_click=increment)
st.button("- Decrement", on_click=decrement)
st.metric("Count", st.session_state.count)
```

### Callback with Arguments

```python
def change_by(amount):
    st.session_state.count += amount

st.button("+5", on_click=change_by, args=(5,))
st.button("-3", on_click=change_by, args=(-3,))

# Using kwargs
def update_name(new_name):
    st.session_state.name = new_name

st.button("Set Alice", on_click=update_name, kwargs={"new_name": "Alice"})
```

### on_change for Input Widgets

```python
def on_text_change():
    st.session_state.last_input = st.session_state.search_query

query = st.text_input("Search", key="search_query", on_change=on_text_change)
```

### Callback for Validation

```python
import streamlit as st

def validate_age():
    if st.session_state.age_input < 0:
        st.session_state.age_input = 0
    elif st.session_state.age_input > 150:
        st.session_state.age_input = 150

st.number_input("Age", min_value=0, max_value=150, key="age_input", on_change=validate_age)
```

---

## Forms

Forms batch multiple widget interactions together. The app only reruns when the **Submit button** is clicked, not on each individual widget change. This is useful for multi-field input where you want atomic submission.

```python
import streamlit as st
import datetime

with st.form("user_form"):
    st.subheader("User Registration")
    name = st.text_input("Full name")
    email = st.text_input("Email")
    age = st.number_input("Age", min_value=0, max_value=120)
    dob = st.date_input("Date of birth", value=datetime.date(1990, 1, 1))
    agree = st.checkbox("I agree to the terms")

    submitted = st.form_submit_button("Register", type="primary")

if submitted:
    if not name or not email:
        st.error("Name and email are required.")
    elif not agree:
        st.warning("You must accept the terms.")
    else:
        st.success(f"Registered {name} ({email})!")
```

### Form Parameters

```python
with st.form(
    key="my_form",           # Unique key (required)
    clear_on_submit=True,    # Reset widgets after submit
    enter_to_submit=True,    # Allow Enter key to submit
    border=True,             # Show form border
):
    ...
    st.form_submit_button("Submit")
```

### Form with Callback

Only the submit button can have a callback inside a form:

```python
import streamlit as st

if "form_data" not in st.session_state:
    st.session_state.form_data = {}

def process_form():
    st.session_state.form_data = {
        "name": st.session_state.form_name,
        "score": st.session_state.form_score,
    }

with st.form("scored_form"):
    st.text_input("Name", key="form_name")
    st.slider("Score", 0, 100, 50, key="form_score")
    st.form_submit_button("Submit", on_click=process_form)

if st.session_state.form_data:
    st.json(st.session_state.form_data)
```

### Multi-step Form Pattern

```python
import streamlit as st

if "step" not in st.session_state:
    st.session_state.step = 1
if "form_data" not in st.session_state:
    st.session_state.form_data = {}

step = st.session_state.step

if step == 1:
    with st.form("step1"):
        st.subheader("Step 1: Personal Info")
        name = st.text_input("Name")
        if st.form_submit_button("Next →"):
            st.session_state.form_data["name"] = name
            st.session_state.step = 2
            st.rerun()

elif step == 2:
    with st.form("step2"):
        st.subheader("Step 2: Preferences")
        lang = st.selectbox("Language", ["Python", "JavaScript", "Rust"])
        if st.form_submit_button("Submit"):
            st.session_state.form_data["language"] = lang
            st.session_state.step = 3
            st.rerun()

elif step == 3:
    st.success("Registration complete!")
    st.json(st.session_state.form_data)
    if st.button("Start over"):
        st.session_state.step = 1
        st.session_state.form_data = {}
        st.rerun()

st.progress(step / 3, text=f"Step {step} of 3")
```

---

## st.cache_data

`@st.cache_data` caches functions that return **serializable data** (DataFrames, lists, dicts, strings, numbers). Each caller gets their own copy of the cached result, making it safe for mutable data.

```python
import streamlit as st
import pandas as pd
import time

@st.cache_data
def load_data(url: str) -> pd.DataFrame:
    """Expensive function: runs once, cached forever (until cache is cleared)."""
    return pd.read_csv(url)

@st.cache_data(ttl=300)  # Cache expires after 5 minutes
def fetch_live_data(endpoint: str) -> dict:
    import requests
    return requests.get(endpoint).json()

@st.cache_data(max_entries=10)  # Keep max 10 cached results
def compute_embedding(text: str) -> list:
    time.sleep(2)  # Simulate expensive computation
    return [0.1, 0.2, 0.3]  # Fake embedding

# Usage
df = load_data("https://raw.githubusercontent.com/mwaskom/seaborn-data/master/iris.csv")
st.dataframe(df)
```

### Excluding Parameters from Cache Key

Prefix parameter with `_` to exclude it from the cache key:

```python
@st.cache_data
def query_database(_db_connection, query: str) -> pd.DataFrame:
    # _db_connection is excluded from the cache key
    return pd.read_sql(query, _db_connection)
```

### Custom Hash Functions

```python
@st.cache_data(hash_funcs={pd.DataFrame: lambda df: df.shape})
def process(df: pd.DataFrame) -> pd.DataFrame:
    return df * 2
```

### Manual Cache Invalidation

```python
@st.cache_data
def get_user_data(user_id: int) -> dict:
    return fetch_from_db(user_id)

# Clear all caches for this function
if st.button("Refresh data"):
    get_user_data.clear()
    st.rerun()

# Clear all caches globally
st.cache_data.clear()
```

---

## st.cache_resource

`@st.cache_resource` caches **global resources** that should be shared across all users and reruns: database connections, ML models, file handles. Returns the **same object** (not a copy), so be careful with mutation.

```python
import streamlit as st
from transformers import pipeline
import sqlite3

@st.cache_resource
def load_model():
    """Loaded once, shared across all users and sessions."""
    return pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")

@st.cache_resource
def get_db_connection():
    return sqlite3.connect("app.db", check_same_thread=False)

# Usage
model = load_model()
conn = get_db_connection()

text = st.text_input("Enter text for sentiment analysis")
if text:
    result = model(text)
    st.write(result)
```

### Thread Safety Warning

Since all users share the same cached object, protect mutable operations:

```python
import threading

@st.cache_resource
def get_counter():
    return {"value": 0, "lock": threading.Lock()}

counter = get_counter()
with counter["lock"]:
    counter["value"] += 1
st.write(f"Total visits: {counter['value']}")
```

---

## Cache Parameters Reference

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ttl` | int/float/timedelta/str | None | Time-to-live before cache expires. Accepts `"1d"`, `"2h30m"`, `3600`, `timedelta(hours=1)` |
| `max_entries` | int | None | Max number of cached entries; oldest is evicted when full |
| `show_spinner` | bool/str | True | Show spinner during computation. Pass a string for custom message |
| `hash_funcs` | dict | None | Custom hash functions for unhashable types: `{MyClass: hash_fn}` |
| `validate` | callable | None | Function called on each cache hit; returns `True` to use cache, `False` to recompute |
| `experimental_allow_widgets` | bool | False | Allow widget calls inside cached function |

```python
@st.cache_data(
    ttl="1h",
    max_entries=100,
    show_spinner="Loading from database...",
)
def expensive_query(sql: str) -> pd.DataFrame:
    return pd.read_sql(sql, get_db_connection())
```

### When to Use Which Cache

| Scenario | Use |
|----------|-----|
| Load CSV / parquet file | `@st.cache_data` |
| API call returning JSON | `@st.cache_data` |
| Data transformation | `@st.cache_data` |
| ML model inference | `@st.cache_data` |
| Load ML model weights | `@st.cache_resource` |
| Database connection | `@st.cache_resource` |
| HTTP session/client | `@st.cache_resource` |

---

## Fragments

`@st.fragment` allows a function to rerun independently without triggering a full app rerun. Introduced in Streamlit 1.37.0.

```python
import streamlit as st
import time

@st.fragment
def live_metrics():
    """Only this section reruns when its widgets change."""
    st.subheader("Live Metrics")
    metric = st.selectbox("Metric", ["CPU", "Memory", "Requests"], key="metric_select")
    value = get_metric(metric)
    st.metric(metric, value)

@st.fragment
def data_table():
    """This section reruns independently."""
    st.subheader("Data Table")
    rows = st.slider("Rows", 5, 100, 20, key="rows_slider")
    st.dataframe(generate_data(rows))

# Both fragments are independent; interacting with one doesn't rerun the other
live_metrics()
data_table()
st.write("This main area only reruns on full reruns.")
```

### Auto-refreshing Fragments

```python
@st.fragment(run_every="5s")  # Rerun every 5 seconds
def streaming_chart():
    data = fetch_latest_data()
    st.line_chart(data)

streaming_chart()
```

`run_every` accepts: seconds (int/float), `"30s"`, `"1m"`, `"1h"`, `timedelta`.

### Fragment in Containers

```python
@st.fragment
def sidebar_widget():
    query = st.text_input("Filter", key="filter")
    return query

with st.sidebar:
    sidebar_widget()
```

### Fragment Communication via Session State

```python
import streamlit as st

if "selected_row" not in st.session_state:
    st.session_state.selected_row = None

@st.fragment
def data_picker():
    df = load_data()
    event = st.dataframe(df, on_select="rerun", selection_mode="single-row")
    if event.selection.rows:
        st.session_state.selected_row = event.selection.rows[0]

@st.fragment
def detail_panel():
    if st.session_state.selected_row is not None:
        row = load_data().iloc[st.session_state.selected_row]
        st.json(row.to_dict())
    else:
        st.info("Select a row to see details.")

col1, col2 = st.columns([2, 1])
with col1:
    data_picker()
with col2:
    detail_panel()
```

---

## Patterns & Best Practices

### Initialize All State Upfront

```python
import streamlit as st

def init_state():
    defaults = {
        "logged_in": False,
        "username": "",
        "cart": [],
        "page": "home",
    }
    for key, val in defaults.items():
        if key not in st.session_state:
            st.session_state[key] = val

init_state()  # Call at app start
```

### State Machine Pattern

```python
import streamlit as st

if "app_state" not in st.session_state:
    st.session_state.app_state = "idle"

state = st.session_state.app_state

if state == "idle":
    if st.button("Start"):
        st.session_state.app_state = "running"
        st.rerun()

elif state == "running":
    with st.spinner("Processing..."):
        result = run_long_task()
        st.session_state.result = result
        st.session_state.app_state = "done"
    st.rerun()

elif state == "done":
    st.success("Complete!")
    st.write(st.session_state.result)
    if st.button("Restart"):
        st.session_state.app_state = "idle"
        st.rerun()
```

### Serializable Session State (Config)

For deployment environments requiring session state serialization:

```toml
# .streamlit/config.toml
[runner]
enforceSerializableSessionState = true
```

This restricts session state to pickle-serializable objects (no lambdas, no module references).
