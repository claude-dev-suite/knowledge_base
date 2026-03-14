# Streamlit - Basics

> Official Documentation: https://docs.streamlit.io/get-started/fundamentals/main-concepts

## Overview

Streamlit is an open-source Python framework for building interactive web applications for data science and machine learning with minimal code. It reruns the entire Python script from top to bottom on every user interaction, making the mental model simple: write Python, get an app.

---

## Table of Contents

1. [Installation](#installation)
2. [Running an App](#running-an-app)
3. [App Architecture & Data Flow](#app-architecture--data-flow)
4. [st.write — Universal Display](#stwrite--universal-display)
5. [Magic Commands](#magic-commands)
6. [Text & Heading Functions](#text--heading-functions)
7. [Markdown](#markdown)
8. [Code Blocks & LaTeX](#code-blocks--latex)
9. [Page Configuration](#page-configuration)
10. [Status & Feedback Elements](#status--feedback-elements)

---

## Installation

```bash
pip install streamlit
```

Verify installation:

```bash
streamlit hello
```

This launches the built-in demo app at `http://localhost:8501`.

---

## Running an App

```bash
streamlit run your_app.py
```

Run with custom arguments (must follow `--`):

```bash
streamlit run your_app.py -- --my-arg value
```

Run as a Python module:

```bash
python -m streamlit run your_app.py
```

Streamlit launches a local server and opens your default browser. The default port is `8501`. To specify a port:

```bash
streamlit run your_app.py --server.port 8080
```

---

## App Architecture & Data Flow

Streamlit's execution model is unique:

- The **entire script reruns** from top to bottom whenever:
  - You save changes to the source file (prompts "Rerun" or "Always rerun")
  - A user interacts with any widget

- **Session State** is the only way to persist values across reruns (see `state.md`)
- **Caching** (`@st.cache_data`, `@st.cache_resource`) prevents expensive operations from re-running (see `state.md`)

Minimal app structure:

```python
import streamlit as st

st.title("My App")
st.write("Hello, world!")
```

A typical app layout:

```python
import streamlit as st
import pandas as pd

# 1. Page config (must be first Streamlit call)
st.set_page_config(page_title="My App", layout="wide")

# 2. Sidebar controls
with st.sidebar:
    option = st.selectbox("Choose dataset", ["iris", "titanic"])

# 3. Main content
st.title("Data Explorer")
df = pd.read_csv(f"data/{option}.csv")
st.dataframe(df)
```

---

## st.write — Universal Display

`st.write()` is the Swiss Army knife of Streamlit. It accepts almost anything and renders it appropriately.

```python
import streamlit as st
import pandas as pd
import numpy as np

# String / Markdown
st.write("This is **bold** and _italic_")

# Numbers
st.write(42)
st.write(3.14)

# DataFrame
df = pd.DataFrame({"a": [1, 2, 3], "b": [4, 5, 6]})
st.write(df)

# Dict / JSON
st.write({"name": "Alice", "age": 30})

# Multiple arguments at once
st.write("The shape is:", df.shape)

# Matplotlib figure
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot([1, 2, 3], [4, 5, 6])
st.write(fig)
```

`st.write` dispatches to the most appropriate specialized function based on the argument type.

### st.write_stream

Stream output token by token (great for LLM responses):

```python
import time

def stream_response():
    for word in ["Hello", " world", " from", " Streamlit"]:
        yield word
        time.sleep(0.1)

st.write_stream(stream_response())
```

---

## Magic Commands

Any expression on its own line in a Streamlit script is automatically displayed using `st.write()`:

```python
import streamlit as st
import pandas as pd

# These are equivalent to calling st.write(...)
"# My Title"         # Rendered as Markdown heading
42                   # Rendered as number
[1, 2, 3]            # Rendered as list

df = pd.DataFrame({"x": [1, 2], "y": [3, 4]})
df                   # Rendered as interactive table
```

Magic works for: strings, numbers, DataFrames, dicts, lists, Matplotlib/Altair/Plotly figures, and more.

---

## Text & Heading Functions

### st.title

```python
st.title("My Application Title")
st.title("Title with anchor", anchor="custom-anchor")
st.title("Hide anchor", anchor=False)
```

### st.header

```python
st.header("Section Header")
st.header("With Divider", divider=True)
st.header("Colored Divider", divider="blue")  # red, orange, green, blue, violet, gray, rainbow
```

### st.subheader

```python
st.subheader("Subsection")
st.subheader("With Divider", divider="gray")
```

### st.text

Fixed-width, preformatted text (no Markdown rendering):

```python
st.text("Hello World")
st.text("Fixed width: useful for logs or ASCII art")
```

### st.caption

Small, muted text for footnotes or metadata:

```python
st.caption("Photo by Alice — CC BY 4.0")
st.caption("_Italic caption_ with **bold**")
```

### st.divider

Horizontal rule:

```python
st.write("Above divider")
st.divider()
st.write("Below divider")
```

---

## Markdown

`st.markdown()` renders Markdown text including GitHub Flavored Markdown:

```python
st.markdown("# Heading 1")
st.markdown("## Heading 2")
st.markdown("**bold**, _italic_, `code`")
st.markdown("[Link](https://streamlit.io)")
st.markdown("- Item 1\n- Item 2\n- Item 3")
st.markdown("> Blockquote text")
```

Unsafe HTML (disabled by default):

```python
st.markdown("<b>Bold HTML</b>", unsafe_allow_html=True)
st.markdown(
    '<p style="color:red;">Red text</p>',
    unsafe_allow_html=True
)
```

Multi-line markdown with triple quotes:

```python
st.markdown("""
## My Report

This analysis covers:
- **Revenue**: $1.2M
- **Users**: 45,000
- **Growth**: 12% MoM

> Key insight: Q3 outperformed forecast by 8%.
""")
```

---

## Code Blocks & LaTeX

### st.code

```python
st.code("import streamlit as st", language="python")

st.code("""
def greet(name: str) -> str:
    return f"Hello, {name}!"
""", language="python")

# Other supported languages
st.code("SELECT * FROM users WHERE active = 1", language="sql")
st.code('{ "key": "value" }', language="json")
st.code("# comment\nkey: value", language="yaml")
```

Line numbers can be shown:

```python
st.code("line1\nline2\nline3", language="python", line_numbers=True)
```

### st.latex

Render LaTeX mathematical expressions:

```python
st.latex(r"\int_a^b f(x)\,dx = F(b) - F(a)")
st.latex(r"\sum_{n=1}^{\infty} \frac{1}{n^2} = \frac{\pi^2}{6}")
st.latex(r"E = mc^2")
```

Inline LaTeX in Markdown (using `$`):

```python
st.markdown("The formula is $E = mc^2$ — elegant.")
```

---

## Page Configuration

`st.set_page_config()` **must be the first Streamlit command** in your script (before any other `st.*` call):

```python
import streamlit as st

st.set_page_config(
    page_title="My App",          # Browser tab title
    page_icon="🚀",               # Favicon (emoji, :material/icon_name:, or image)
    layout="wide",                # "centered" (default) or "wide"
    initial_sidebar_state="expanded",  # "auto", "expanded", "collapsed"
    menu_items={
        "Get Help": "https://docs.myapp.com",
        "Report a bug": "https://github.com/myorg/myapp/issues",
        "About": "## My App\nVersion 1.0.0"
    }
)

st.title("Welcome")
```

Layout options:
- `"centered"` — content in a fixed-width centered column (default)
- `"wide"` — content uses full browser width

Page icon accepts:
- Emoji: `"🦈"`
- Emoji shortcode: `":shark:"`
- Material icon: `":material/thumb_up:"`
- Random emoji: `"random"`

---

## Status & Feedback Elements

### st.spinner

```python
import time

with st.spinner("Loading data..."):
    time.sleep(2)
    data = load_heavy_data()

st.success("Data loaded!")
```

### Alert Messages

```python
st.success("Operation completed successfully!")
st.info("Here is some information.")
st.warning("This might cause issues.")
st.error("Something went wrong.")
```

With icons:

```python
st.success("Done!", icon="✅")
st.error("Failed!", icon="🚨")
```

### st.exception

```python
try:
    result = risky_operation()
except Exception as e:
    st.exception(e)  # Shows full traceback in the app
```

### st.progress

```python
import time

bar = st.progress(0)
for i in range(100):
    time.sleep(0.01)
    bar.progress(i + 1)
st.success("Done!")
```

With text:

```python
bar = st.progress(0, text="Processing...")
bar.progress(50, text="Halfway there...")
bar.progress(100, text="Complete!")
```

### st.toast

Non-blocking toast notification:

```python
st.toast("Saved!", icon="💾")
st.toast("Warning message", icon="⚠️")
```

### st.balloons / st.snow

```python
st.balloons()   # Celebratory balloons
st.snow()       # Snowflakes
```

---

## Complete Minimal App Example

```python
import streamlit as st
import pandas as pd
import numpy as np

st.set_page_config(page_title="Demo", page_icon="📊", layout="wide")

st.title("📊 Data Demo")
st.markdown("A minimal Streamlit app demonstrating core basics.")

# Sidebar
with st.sidebar:
    st.header("Controls")
    n_rows = st.slider("Number of rows", 10, 100, 50)
    seed = st.number_input("Random seed", value=42)

# Generate data
np.random.seed(seed)
df = pd.DataFrame(
    np.random.randn(n_rows, 4),
    columns=["A", "B", "C", "D"]
)

# Display
col1, col2 = st.columns(2)

with col1:
    st.subheader("Raw Data")
    st.dataframe(df)

with col2:
    st.subheader("Statistics")
    st.write(df.describe())

st.line_chart(df)
st.caption(f"Showing {n_rows} rows with seed {seed}")
```
