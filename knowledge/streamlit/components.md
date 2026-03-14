# Streamlit - Components

> Official Documentation: https://docs.streamlit.io/develop/api-reference

## Overview

Streamlit provides a rich set of components divided into three categories: **input widgets** (collect user input), **display elements** (render data and media), and **layout containers** (structure the UI). All widgets return their current value and trigger a full app rerun when changed (unless inside a fragment or form).

---

## Table of Contents

1. [Input Widgets — Buttons](#input-widgets--buttons)
2. [Input Widgets — Text & Numbers](#input-widgets--text--numbers)
3. [Input Widgets — Selection](#input-widgets--selection)
4. [Input Widgets — Date & Time](#input-widgets--date--time)
5. [Input Widgets — Media](#input-widgets--media)
6. [Display — Data](#display--data)
7. [Display — Charts](#display--charts)
8. [Display — Media](#display--media)
9. [Layout — Columns & Tabs](#layout--columns--tabs)
10. [Layout — Sidebar & Containers](#layout--sidebar--containers)
11. [Layout — Expander & Popover](#layout--expander--popover)

---

## Input Widgets — Buttons

### st.button

```python
import streamlit as st

clicked = st.button("Click me")
if clicked:
    st.write("Button was clicked!")

# With icon and help tooltip
st.button("Submit", icon="📤", help="Send the form data", type="primary")

# Disabled button
st.button("Not available", disabled=True)

# use_container_width fills available horizontal space
st.button("Full-width", use_container_width=True)
```

`type` options: `"secondary"` (default), `"primary"`, `"tertiary"`

### st.download_button

```python
import pandas as pd

df = pd.DataFrame({"col1": [1, 2], "col2": [3, 4]})
csv = df.to_csv(index=False).encode("utf-8")

st.download_button(
    label="Download CSV",
    data=csv,
    file_name="data.csv",
    mime="text/csv",
    icon="⬇️"
)

# Download binary file
with open("report.pdf", "rb") as f:
    st.download_button("Download PDF", f, "report.pdf", "application/pdf")
```

### st.link_button

```python
st.link_button("Open Docs", "https://docs.streamlit.io")
st.link_button("GitHub", "https://github.com", icon=":material/open_in_new:")
```

---

## Input Widgets — Text & Numbers

### st.text_input

```python
name = st.text_input("Your name")
st.write(f"Hello, {name}")

# With placeholder, default value, max chars
email = st.text_input(
    "Email address",
    value="",
    placeholder="user@example.com",
    max_chars=100,
    help="We'll never share your email."
)

# Password field
password = st.text_input("Password", type="password")

# Trigger only on Enter (not on every keystroke)
query = st.text_input("Search", on_change=None)
```

### st.text_area

```python
text = st.text_area(
    "Description",
    height=150,
    placeholder="Describe your project...",
    max_chars=500
)
```

### st.number_input

```python
age = st.number_input("Age", min_value=0, max_value=120, value=25, step=1)
price = st.number_input("Price (USD)", min_value=0.0, value=9.99, step=0.01, format="%.2f")
```

### st.slider

```python
# Integer slider
value = st.slider("Number of items", min_value=1, max_value=100, value=50, step=1)

# Float slider
opacity = st.slider("Opacity", 0.0, 1.0, 0.5, 0.05)

# Range slider (returns tuple)
date_range = st.slider(
    "Price range",
    min_value=0,
    max_value=1000,
    value=(100, 500)
)
low, high = date_range

# Date slider
import datetime
date = st.slider(
    "Select date",
    min_value=datetime.date(2020, 1, 1),
    max_value=datetime.date(2025, 12, 31),
    value=datetime.date(2024, 1, 1)
)
```

### st.select_slider

Slider over a discrete list of values:

```python
size = st.select_slider(
    "T-shirt size",
    options=["XS", "S", "M", "L", "XL", "XXL"],
    value="M"
)

# Range select slider
start, end = st.select_slider(
    "Range",
    options=range(1, 11),
    value=(3, 7)
)
```

---

## Input Widgets — Selection

### st.checkbox

```python
agree = st.checkbox("I agree to the terms")
if agree:
    st.write("Thanks for agreeing!")

show_raw = st.checkbox("Show raw data", value=True)  # checked by default
```

### st.toggle

```python
dark_mode = st.toggle("Dark mode", value=False)
if dark_mode:
    st.write("Dark mode enabled")
```

### st.radio

```python
color = st.radio(
    "Favorite color",
    options=["Red", "Green", "Blue"],
    index=0,            # default selection
    horizontal=True     # display horizontally
)
st.write(f"You chose: {color}")

# With captions
genre = st.radio(
    "Genre",
    options=["Comedy", "Drama", "Sci-Fi"],
    captions=["Laugh out loud", "Deep stories", "Future worlds"]
)
```

### st.selectbox

```python
country = st.selectbox(
    "Country",
    options=["Italy", "France", "Germany", "Spain"],
    index=0,
    placeholder="Choose a country"
)

# With format_func for display transformation
status_code = st.selectbox(
    "Status",
    options=[200, 404, 500],
    format_func=lambda x: f"{x} — {'OK' if x == 200 else 'Error'}"
)
```

### st.multiselect

```python
fruits = st.multiselect(
    "Select fruits",
    options=["Apple", "Banana", "Cherry", "Date", "Elderberry"],
    default=["Apple", "Cherry"]
)
st.write(f"Selected: {fruits}")

# Max selections
tags = st.multiselect("Tags (max 3)", options=["A", "B", "C", "D", "E"], max_selections=3)
```

### st.color_picker

```python
bg_color = st.color_picker("Background color", "#00BFFF")
st.markdown(
    f'<div style="background-color:{bg_color};padding:20px">Preview</div>',
    unsafe_allow_html=True
)
```

---

## Input Widgets — Date & Time

### st.date_input

```python
import datetime

birthday = st.date_input("Birthday", value=datetime.date(1990, 1, 1))

# Date range (returns tuple when value is a tuple)
date_range = st.date_input(
    "Booking period",
    value=(datetime.date.today(), datetime.date.today() + datetime.timedelta(days=7)),
    min_value=datetime.date.today(),
    format="DD/MM/YYYY"    # display format
)
```

### st.time_input

```python
import datetime

meeting_time = st.time_input("Meeting time", value=datetime.time(9, 0))
step_time = st.time_input("Alarm", step=900)  # 15-minute steps (in seconds)
```

---

## Input Widgets — Media

### st.file_uploader

```python
# Single file
uploaded = st.file_uploader("Upload a CSV", type=["csv"])
if uploaded is not None:
    import pandas as pd
    df = pd.read_csv(uploaded)
    st.dataframe(df)

# Multiple files
files = st.file_uploader("Upload images", type=["jpg", "jpeg", "png"], accept_multiple_files=True)
for f in files:
    st.image(f, caption=f.name)
```

The uploaded file is a file-like object with `.name`, `.size`, `.type` attributes and `.read()` method.

### st.camera_input

```python
photo = st.camera_input("Take a picture")
if photo:
    st.image(photo)
    # Process with PIL
    from PIL import Image
    img = Image.open(photo)
    st.write(f"Size: {img.size}")
```

---

## Display — Data

### st.dataframe

Interactive sortable/scrollable table:

```python
import pandas as pd
import numpy as np

df = pd.DataFrame(np.random.randn(50, 5), columns=list("ABCDE"))

st.dataframe(df)

# With height and width constraints
st.dataframe(df, height=200, use_container_width=True)

# With highlighting
st.dataframe(df.style.highlight_max(axis=0))

# Column configuration
st.dataframe(
    df,
    column_config={
        "A": st.column_config.NumberColumn("Column A", format="%.3f"),
        "B": st.column_config.ProgressColumn("B", min_value=-3, max_value=3),
    }
)
```

### st.data_editor

Editable dataframe:

```python
edited_df = st.data_editor(
    df,
    num_rows="dynamic",      # Allow adding/deleting rows
    use_container_width=True,
    column_config={
        "A": st.column_config.NumberColumn(min_value=0, max_value=100),
    }
)
st.write("Edited data:", edited_df)
```

### st.table

Static (non-interactive) table:

```python
st.table(df.head(10))
```

### st.metric

Large KPI-style metric display:

```python
st.metric(label="Revenue", value="$1.2M", delta="$200K", delta_color="normal")
st.metric("Users", 42_000, -500, delta_color="inverse")  # inverse: red=good, green=bad
st.metric("Uptime", "99.9%", delta=None)

# In columns
col1, col2, col3 = st.columns(3)
col1.metric("Temperature", "70 °F", "1.2 °F")
col2.metric("Wind", "9 mph", "-8%")
col3.metric("Humidity", "86%", "4%")
```

### st.json

Pretty-printed JSON display:

```python
data = {"name": "Alice", "scores": [98, 87, 92], "active": True}
st.json(data)
st.json(data, expanded=False)  # Collapsed by default
```

---

## Display — Charts

### Native Streamlit Charts

```python
import pandas as pd
import numpy as np

df = pd.DataFrame(np.random.randn(20, 3), columns=["A", "B", "C"])

st.line_chart(df)
st.area_chart(df)
st.bar_chart(df)
st.scatter_chart(df, x="A", y="B", color="C", size="C")
```

With explicit columns:

```python
st.line_chart(df, x="A", y=["B", "C"], color=["#FF0000", "#00FF00"])
```

### st.map

```python
import pandas as pd

# Requires 'lat' and 'lon' columns
map_df = pd.DataFrame({
    "lat": [37.76, 37.78, 37.74],
    "lon": [-122.4, -122.41, -122.42],
    "size": [100, 200, 150]
})
st.map(map_df, latitude="lat", longitude="lon", size="size", zoom=12)
```

### st.pyplot

```python
import matplotlib.pyplot as plt
import numpy as np

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(np.random.randn(100).cumsum(), label="Random Walk")
ax.set_title("Cumulative Sum")
ax.legend()
st.pyplot(fig)
plt.close(fig)  # Important: free memory
```

### st.altair_chart

```python
import altair as alt
import pandas as pd

df = pd.DataFrame({"x": range(20), "y": range(20), "category": ["A"] * 10 + ["B"] * 10})

chart = alt.Chart(df).mark_circle().encode(
    x="x", y="y", color="category", size=alt.value(100)
)
st.altair_chart(chart, use_container_width=True)
```

### st.plotly_chart

```python
import plotly.express as px

df = px.data.iris()
fig = px.scatter(df, x="sepal_width", y="sepal_length", color="species")
st.plotly_chart(fig, use_container_width=True)
```

---

## Display — Media

### st.image

```python
# From URL
st.image("https://example.com/photo.jpg", caption="Example image", use_container_width=True)

# From file path
st.image("./assets/logo.png", width=200)

# From numpy array (e.g., OpenCV frame)
import numpy as np
img_array = np.random.randint(0, 255, (100, 200, 3), dtype=np.uint8)
st.image(img_array, caption="Generated image")

# Multiple images
st.image(["img1.jpg", "img2.jpg"], caption=["First", "Second"])
```

### st.video

```python
st.video("https://www.youtube.com/watch?v=dQw4w9WgXcQ")
st.video("./local_video.mp4", start_time=30)  # start at 30s
```

### st.audio

```python
st.audio("./audio.mp3")
st.audio(audio_bytes, format="audio/wav")
```

---

## Layout — Columns & Tabs

### st.columns

```python
col1, col2 = st.columns(2)
col1.write("Left column")
col2.write("Right column")

# Context manager style (preferred)
col1, col2 = st.columns(2)
with col1:
    st.header("A")
    st.image("img1.jpg")
with col2:
    st.header("B")
    st.image("img2.jpg")

# Custom width ratios
left, mid, right = st.columns([1, 2, 1])

# With gap
c1, c2 = st.columns(2, gap="large")  # "small", "medium", "large"

# Vertical alignment
top, mid, bot = st.columns(3, vertical_alignment="center")
```

### st.tabs

```python
tab1, tab2, tab3 = st.tabs(["Overview", "Data", "Settings"])

with tab1:
    st.write("Overview content")
    st.metric("Users", 42_000)

with tab2:
    st.dataframe(df)

with tab3:
    theme = st.selectbox("Theme", ["Light", "Dark"])
```

---

## Layout — Sidebar & Containers

### st.sidebar

```python
# Using with
with st.sidebar:
    st.title("Navigation")
    page = st.radio("Go to", ["Home", "Data", "Settings"])
    st.divider()
    st.caption("Version 1.0.0")

# Direct dot notation
st.sidebar.header("Controls")
n = st.sidebar.slider("Count", 1, 100, 50)
```

### st.container

Groups elements; allows out-of-order insertion:

```python
# Container created early, filled later
container = st.container()
st.write("This appears SECOND in the code but AFTER container content")

with container:
    st.write("This appears FIRST because it's in the container above")

# Border
with st.container(border=True):
    st.write("Bordered box")
    st.metric("Value", 42)

# Height-limited scrollable container
with st.container(height=300):
    for i in range(50):
        st.write(f"Row {i}")
```

### st.empty

Single-element placeholder for dynamic updates:

```python
import time

status = st.empty()
for i in range(5):
    status.write(f"Step {i+1}/5 running...")
    time.sleep(1)
status.success("All done!")

# Clear the placeholder
status.empty()
```

---

## Layout — Expander & Popover

### st.expander

Collapsible section to save vertical space:

```python
with st.expander("See explanation", expanded=False):
    st.write("Here is a detailed explanation that is hidden by default.")
    st.image("./diagram.png")

with st.expander("Advanced settings", expanded=True, icon="⚙️"):
    learning_rate = st.slider("Learning rate", 0.0001, 0.1, 0.001, format="%.4f")
    epochs = st.number_input("Epochs", 1, 1000, 100)
```

### st.popover

Floating popover panel triggered by a button:

```python
with st.popover("Open settings"):
    st.markdown("**Display Settings**")
    theme = st.radio("Theme", ["Light", "Dark", "Auto"])
    font_size = st.slider("Font size", 10, 24, 14)
```

### st.dialog

Modal dialog:

```python
@st.dialog("Confirm deletion")
def confirm_dialog(item_name):
    st.write(f"Are you sure you want to delete **{item_name}**?")
    if st.button("Yes, delete", type="primary"):
        delete_item(item_name)
        st.rerun()
    if st.button("Cancel"):
        st.rerun()

if st.button("Delete item"):
    confirm_dialog("Project Alpha")
```

---

## Complete Widget Gallery Example

```python
import streamlit as st
import pandas as pd
import numpy as np

st.set_page_config(page_title="Widget Gallery", layout="wide")
st.title("Streamlit Widget Gallery")

# Sidebar
with st.sidebar:
    st.header("Input Panel")
    name = st.text_input("Name", "Alice")
    age = st.slider("Age", 0, 100, 25)
    color = st.color_picker("Favorite color", "#FF5733")
    uploaded = st.file_uploader("Upload CSV", type="csv")

# Main area tabs
tab1, tab2, tab3 = st.tabs(["Profile", "Data", "Charts"])

with tab1:
    col1, col2 = st.columns(2)
    with col1:
        st.metric("Name", name)
        st.metric("Age", age)
    with col2:
        st.write(f"Color preview:")
        st.markdown(f'<div style="width:80px;height:40px;background:{color};border-radius:8px"></div>', unsafe_allow_html=True)

with tab2:
    if uploaded:
        df = pd.read_csv(uploaded)
        st.dataframe(df, use_container_width=True)
    else:
        df = pd.DataFrame(np.random.randn(20, 3), columns=["X", "Y", "Z"])
        st.data_editor(df, use_container_width=True)

with tab3:
    st.line_chart(df)
    st.bar_chart(df)
```
