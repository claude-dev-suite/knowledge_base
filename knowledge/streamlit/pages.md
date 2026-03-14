# Streamlit - Multipage Apps

> Official Documentation: https://docs.streamlit.io/develop/concepts/multipage-apps/overview

## Overview

Streamlit supports multipage applications through two approaches: the modern **`st.Page` + `st.navigation`** API (recommended, maximum flexibility) and the simpler **`pages/` directory** convention (automatic, less code). Pages share session state when navigated via Streamlit's built-in navigation, but create a new session when navigated via URL.

---

## Table of Contents

1. [Approach 1: st.Page + st.navigation](#approach-1-stpage--stnavigation)
2. [Approach 2: pages/ Directory](#approach-2-pages-directory)
3. [Page Configuration (st.set_page_config)](#page-configuration-stset_page_config)
4. [Navigation Patterns](#navigation-patterns)
5. [Programmatic Navigation](#programmatic-navigation)
6. [Passing Data Between Pages](#passing-data-between-pages)
7. [Dynamic Navigation & Access Control](#dynamic-navigation--access-control)
8. [URL & Deep Linking](#url--deep-linking)

---

## Approach 1: st.Page + st.navigation

This is the **recommended approach** for full control over navigation, page metadata, and shared layout.

### Directory Structure

```
my_app/
├── app.py              # Entrypoint: defines navigation
├── pages/
│   ├── home.py
│   ├── data_explorer.py
│   └── settings.py
└── utils/
    └── helpers.py
```

### Entrypoint File (`app.py`)

The entrypoint acts as a "frame" around all pages — anything you render here appears on every page:

```python
import streamlit as st

# Page config belongs here — runs before any page content
st.set_page_config(page_title="My App", page_icon="🚀", layout="wide")

# Shared elements (e.g., header, sidebar)
st.sidebar.title("My App")
st.sidebar.caption("v1.0.0")

# Define navigation
pg = st.navigation([
    st.Page("pages/home.py", title="Home", icon="🏠", default=True),
    st.Page("pages/data_explorer.py", title="Data Explorer", icon=":material/bar_chart:"),
    st.Page("pages/settings.py", title="Settings", icon="⚙️"),
])

# Run the selected page
pg.run()
```

### Page File (`pages/home.py`)

```python
import streamlit as st

st.title("Welcome Home")
st.write("This is the home page.")
```

### st.Page Parameters

```python
st.Page(
    page,               # str (file path) or callable
    *,
    title=None,         # str — display name in navigation menu
    icon=None,          # emoji, ":material/icon_name:", or "spinner"
    url_path=None,      # str — relative URL path (no leading slash)
    default=False,      # bool — landing page (only one per app)
    visibility="visible"  # "visible" or "hidden"
)
```

### st.navigation Parameters

```python
st.navigation(
    pages,              # list[StreamlitPage] or dict[str, list[StreamlitPage]]
    *,
    position="sidebar", # "sidebar" or "hidden"
    expanded=True       # bool — expand section headers in sidebar
)
```

Returns the currently selected `StreamlitPage` object; call `.run()` on it to execute.

### Grouped Navigation

Use a dict to group pages under section headers:

```python
pg = st.navigation({
    "Overview": [
        st.Page("pages/dashboard.py", title="Dashboard", icon="📊"),
        st.Page("pages/summary.py", title="Summary", icon="📋"),
    ],
    "Management": [
        st.Page("pages/users.py", title="Users", icon="👥"),
        st.Page("pages/settings.py", title="Settings", icon="⚙️"),
    ]
})
pg.run()
```

### Callable Pages (inline functions)

```python
def about_page():
    st.title("About")
    st.write("Built with Streamlit")

pg = st.navigation([
    st.Page("pages/home.py", title="Home", default=True),
    st.Page(about_page, title="About", icon="ℹ️"),
])
pg.run()
```

---

## Approach 2: pages/ Directory

The simpler convention for small apps: place Python files in a `pages/` subdirectory next to your entrypoint.

### Directory Structure

```
my_app/
├── Home.py             # Entrypoint / main page
└── pages/
    ├── 1_Data.py
    ├── 2_Analysis.py
    └── 3_Settings.py
```

### Entrypoint (`Home.py`)

```python
import streamlit as st

st.set_page_config(page_title="My App")
st.title("Home")
st.write("Welcome to the app!")
```

### Page File (`pages/1_Data.py`)

```python
import streamlit as st

st.title("Data Page")
df = load_data()
st.dataframe(df)
```

Streamlit automatically:
- Creates a navigation menu in the sidebar
- Sets page labels from filenames (underscores become spaces, numeric prefix stripped)
- Orders pages by numeric prefix

### Filename Conventions

| Filename | Displayed Label |
|----------|----------------|
| `Data.py` | Data |
| `Data_Explorer.py` | Data Explorer |
| `1_Data_Explorer.py` | Data Explorer |
| `02_Settings_Page.py` | Settings Page |

---

## Page Configuration

`st.set_page_config()` must be called once, as the **first Streamlit command** in the entrypoint file:

```python
import streamlit as st

st.set_page_config(
    page_title="Analytics Dashboard",
    page_icon=":material/analytics:",
    layout="wide",                      # "centered" or "wide"
    initial_sidebar_state="expanded",   # "auto", "expanded", "collapsed"
    menu_items={
        "Get Help": "https://docs.example.com",
        "Report a bug": "https://github.com/org/repo/issues",
        "About": "## Dashboard v2.0\nPowered by Streamlit."
    }
)
```

With `st.Page` + `st.navigation`, individual pages can override the **title** and **icon** (but not the full config) through the `title` and `icon` parameters of `st.Page`.

---

## Navigation Patterns

### Hidden Navigation (Custom Sidebar)

Build a fully custom sidebar navigation using `st.page_link`:

```python
# app.py
import streamlit as st

st.set_page_config(layout="wide")

pg = st.navigation(
    [
        st.Page("pages/home.py"),
        st.Page("pages/data.py"),
        st.Page("pages/settings.py"),
    ],
    position="hidden"  # Don't show default sidebar navigation
)

# Custom sidebar
with st.sidebar:
    st.image("./logo.png", width=120)
    st.title("Navigation")
    st.page_link("pages/home.py", label="Home", icon="🏠")
    st.page_link("pages/data.py", label="Data", icon="📊")
    st.page_link("pages/settings.py", label="Settings", icon="⚙️")
    st.divider()
    if st.button("Logout"):
        logout()

pg.run()
```

### Top Navigation Bar Pattern

```python
# app.py
import streamlit as st

pg = st.navigation([
    st.Page("pages/home.py", title="Home"),
    st.Page("pages/data.py", title="Data"),
    st.Page("pages/reports.py", title="Reports"),
], position="hidden")

# Render navigation as horizontal buttons
cols = st.columns(3)
pages = ["pages/home.py", "pages/data.py", "pages/reports.py"]
labels = ["🏠 Home", "📊 Data", "📈 Reports"]
for col, page, label in zip(cols, pages, labels):
    col.page_link(page, label=label, use_container_width=True)

st.divider()
pg.run()
```

---

## Programmatic Navigation

### st.switch_page

Navigate to another page programmatically (triggers a full rerun of the target page):

```python
import streamlit as st

if not st.session_state.get("logged_in"):
    st.switch_page("pages/login.py")

st.title("Protected Dashboard")
```

Note: `st.switch_page` does NOT preserve session state (it starts a fresh session). Use it only for redirects where state preservation is not needed, or ensure state is set before switching.

### Conditional Navigation (Login Flow)

```python
# app.py
import streamlit as st

if "authenticated" not in st.session_state:
    st.session_state.authenticated = False

def login_page():
    st.title("Login")
    user = st.text_input("Username")
    pwd = st.text_input("Password", type="password")
    if st.button("Login"):
        if user == "admin" and pwd == "secret":
            st.session_state.authenticated = True
            st.session_state.username = user
            st.rerun()
        else:
            st.error("Invalid credentials")

if not st.session_state.authenticated:
    pg = st.navigation([st.Page(login_page, title="Login", default=True)])
else:
    pg = st.navigation([
        st.Page("pages/dashboard.py", title="Dashboard", default=True),
        st.Page("pages/profile.py", title="Profile"),
        st.Page("pages/admin.py", title="Admin"),
    ])

pg.run()
```

---

## Passing Data Between Pages

Session state is the correct way to pass data between pages (when navigating via `st.navigation`/`st.Page`):

```python
# pages/data_loader.py
import streamlit as st
import pandas as pd

st.title("Data Loader")

uploaded = st.file_uploader("Upload CSV")
if uploaded:
    df = pd.read_csv(uploaded)
    st.session_state.shared_df = df
    st.success(f"Loaded {len(df)} rows")
    st.dataframe(df.head())
```

```python
# pages/analysis.py
import streamlit as st

st.title("Analysis")

if "shared_df" not in st.session_state:
    st.warning("No data loaded yet. Please go to 'Data Loader' first.")
    st.page_link("pages/data_loader.py", label="Go to Data Loader →")
else:
    df = st.session_state.shared_df
    st.write(f"Analyzing {len(df)} rows...")
    st.dataframe(df.describe())
```

---

## Dynamic Navigation & Access Control

Show/hide pages based on user roles or conditions:

```python
# app.py
import streamlit as st

# Determine user role
role = st.session_state.get("role", "guest")

# Build page list dynamically
pages = [st.Page("pages/home.py", title="Home", default=True)]

if role in ("user", "admin"):
    pages.append(st.Page("pages/dashboard.py", title="Dashboard"))
    pages.append(st.Page("pages/profile.py", title="My Profile"))

if role == "admin":
    pages.append(st.Page("pages/admin.py", title="Admin Panel", icon="🔒"))

pg = st.navigation(pages)

# Sidebar shows user info
with st.sidebar:
    st.write(f"Role: **{role}**")
    if st.button("Logout"):
        del st.session_state["role"]
        st.rerun()

pg.run()
```

### Hidden Pages (accessible by URL, not in menu)

```python
pg = st.navigation([
    st.Page("pages/home.py", title="Home", default=True),
    st.Page("pages/public.py", title="Public"),
    # Hidden: not in sidebar menu, but accessible via direct URL
    st.Page("pages/debug.py", title="Debug", visibility="hidden"),
])
pg.run()
```

---

## URL & Deep Linking

By default, the URL path is derived from the filename:
- `pages/data_explorer.py` → `/data_explorer`
- `pages/user_settings.py` → `/user_settings`

Override with `url_path`:

```python
st.Page("pages/user_settings.py", title="Settings", url_path="settings")
# → accessible at /settings
```

The default page maps to `/` (root URL).

```python
st.Page("pages/dashboard.py", title="Dashboard", default=True)
# → accessible at / and /dashboard
```

**Important:** Navigating via URL (typing in the browser bar) creates a **new session** and resets all session state. Use `st.navigation` links / `st.page_link` for navigation that preserves session state.

### st.page_link

Renders a clickable link that navigates while preserving session state:

```python
st.page_link("pages/analysis.py", label="Go to Analysis", icon="📈")
st.page_link("pages/analysis.py", label="→ Analyze", disabled=not data_loaded)

# External links
st.page_link("https://streamlit.io", label="Streamlit Docs", icon="🌐")
```

---

## Complete Multipage App Example

### `app.py` (entrypoint)

```python
import streamlit as st

st.set_page_config(
    page_title="Sales Dashboard",
    page_icon=":material/bar_chart:",
    layout="wide"
)

# Shared sidebar content
with st.sidebar:
    st.image("https://via.placeholder.com/150x50?text=Logo", width=150)
    st.divider()

# Navigation
pg = st.navigation({
    "Main": [
        st.Page("pages/overview.py", title="Overview", icon="🏠", default=True),
        st.Page("pages/sales.py", title="Sales", icon=":material/trending_up:"),
    ],
    "Tools": [
        st.Page("pages/upload.py", title="Upload Data", icon="📤"),
        st.Page("pages/export.py", title="Export Report", icon="📥"),
    ]
})

pg.run()
```

### `pages/overview.py`

```python
import streamlit as st
import pandas as pd

st.title("Sales Overview")

if "sales_df" not in st.session_state:
    st.info("No data loaded. Please upload data first.")
    st.page_link("pages/upload.py", label="Upload Data →")
else:
    df = st.session_state.sales_df
    col1, col2, col3 = st.columns(3)
    col1.metric("Total Revenue", f"${df['revenue'].sum():,.0f}")
    col2.metric("Orders", len(df))
    col3.metric("Avg. Order Value", f"${df['revenue'].mean():,.0f}")
    st.line_chart(df.set_index("date")["revenue"])
```

### `pages/upload.py`

```python
import streamlit as st
import pandas as pd

st.title("Upload Sales Data")

f = st.file_uploader("Upload CSV", type="csv")
if f:
    df = pd.read_csv(f)
    st.session_state.sales_df = df
    st.success(f"Loaded {len(df):,} rows")
    st.dataframe(df.head())
    st.page_link("pages/overview.py", label="View Overview →")
```
