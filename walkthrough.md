# Project Walkthrough and Architecture Notes

## 1. Project Overview

This repository is a compact data analytics project built around two primary interfaces:

- a Streamlit dashboard for interactive exploration of tabular data
- a FastAPI backend service for uploading files and extracting summary information

The project is designed to help users work with raw CSV or Excel files, inspect their structure, understand numeric and categorical columns, filter the dataset, and visualize grouped summaries without needing complex manual pandas code.

At a high level, the app answers a simple but practical question:

> "Given any dataset, how can I inspect it quickly, understand its structure, filter it intelligently, and derive useful summary insights?"

The use of pandas makes this possible because it gives the project a flexible DataFrame layer for data cleaning, summarization, aggregation, and filtering.

---

## 2. Why this project exists

This project is a great example of a lightweight analytics utility. It demonstrates how to:

- ingest tabular data from local files or uploaded files
- infer column types automatically
- summarize dataset structure
- filter rows by value or substring
- aggregate numeric data by category
- present insights through a friendly UI

This is useful in real-world tasks such as:

- sales reporting
- exploratory data analysis (EDA)
- quick validation of uploaded spreadsheets
- prototype analytics services for internal teams
- training examples for FastAPI + Streamlit + pandas integration

---

## 3. Project architecture: reason behind the folder layout

```text
pandas-analytics/
├── app/
│   ├── api/
│   │   └── v1/
│   │       ├── router.py
│   │       ├── schemas.py
│   │       └── __init__.py
│   ├── core/
│   │   ├── config.py
│   │   └── __init__.py
│   ├── services/
│   │   ├── analyze_data.py
│   │   └── __init__.py
│   ├── main.py
│   └── __init__.py
├── data/
│   ├── raw/
│   └── processed/
├── notebooks/
│   └── eda.ipynb
├── streamlit_app.py
├── requirements.txt
├── README.md
├── walkthrough.md
└── .gitignore
```

### Why this structure is useful

#### app/
This is the backend application package.

- The project keeps backend logic isolated from the UI.
- This helps prevent mixing UI logic with analytics logic.
- It makes the code easier to scale when more routes and features are added.

#### app/api/
This folder stores the API layer.

- router.py handles endpoints and HTTP logic
- schemas.py defines the shape of request/response data using Pydantic
- This is the right place for route loading, validation, and response modeling

#### app/services/
This stores business logic and pandas operations.

- data loading logic lives here
- filtering and summary functions are separated from the HTTP layer
- this keeps API endpoints thin and clean

#### app/core/
This is intended for shared configuration and app-level settings

- in a larger project, this may hold environment config, app metadata, and settings
- in this repo, it is currently minimal, which is normal for a small prototype

#### data/
This folder is used to hold raw and processed data. It separates:

- raw source files
- cleaned or generated outputs

This keeps the project organized and avoids mixing business data with source code.

#### notebooks/
This folder is meant for interactive exploration and experimentation.

- it is a place for EDA and prototype analysis
- useful when developers want to test ideas before moving logic into production code

#### streamlit_app.py
This acts as the frontend dashboard. It is the main interactive user experience.

---

## 4. End-to-end working flow

### Step 1: Application startup

The app starts in two different ways depending on the interface:

#### Streamlit entry point

```python
st.set_page_config(
    page_title="Universal Data Analytics Studio",
    page_icon="📊",
    layout="wide",
)
```

This configures the browser page title and layout of the dashboard.

The application immediately renders:

- a title
- a caption
- a sidebar with file upload controls
- KPI cards for structural overview

#### FastAPI entry point

```python
app = FastAPI(
    title="Pandas Data Analytics API",
    version="1.0.0",
    description="FastAPI service for dataset upload, summary metrics, and analytics",
)
```

The API app is created as a separate service and includes a router mounted under `/api/v1`.

---

### Step 2: Data ingestion

In the Streamlit app, the project allows a user to upload a CSV or Excel file:

```python
uploaded_file = st.sidebar.file_uploader(
    "Upload a CSV or Excel file", type=["csv", "xlsx", "xls"]
)
```

If a user does not upload a file, the app falls back to the default dataset:

```python
DEFAULT_DATA_PATH = BASE_DIR / "data" / "raw" / "Sales.csv"
```

This is a good design because it allows local testing without requiring a file upload every time.

The actual loading is done in a cached function:

```python
@st.cache_data
def load_data(file_source):
    if hasattr(file_source, "name"):
        if file_source.name.endswith((".xlsx", ".xls")):
            return pd.read_excel(file_source)
        return pd.read_csv(file_source)
    elif str(file_source).endswith((".xlsx", ".xls")):
        return pd.read_excel(file_source)
    return pd.read_csv(file_source)
```

#### Why this matters

- `@st.cache_data` prevents re-reading the same file repeatedly
- the function supports both uploaded objects and local file paths
- it handles CSV and Excel files gracefully

This pattern is a common Streamlit best practice because it reduces overhead and improves responsiveness.

---

### Step 3: Column detection and dataset overview

Once the DataFrame is loaded, the app identifies numeric and categorical columns automatically:

```python
num_cols = df.select_dtypes(include=["number"]).columns.tolist()
cat_cols = df.select_dtypes(
    include=["object", "category", "string"]
).columns.tolist()
```

This is one of the project’s most important design decisions: it does not assume the user’s dataset has fixed column names. Instead, the code works across different tables.

The app then renders summary KPIs:

```python
kpi1.metric("Total Records", f"{df.shape[0]:,}")
kpi2.metric("Total Features", df.shape[1])
kpi3.metric("Numeric Metrics", len(num_cols))
kpi4.metric("Categorical Features", len(cat_cols))
```

This gives the user an immediate overview of the dataset’s structure.

---

### Step 4: Filtering and slicing

The sidebar includes a dynamic slicer that lets users filter by column.

```python
filter_col = st.sidebar.selectbox(
    "Choose Column to Slice:", options=["None"] + df.columns.tolist()
)
```

Then the project reacts based on the data type of that column.

#### If the selected column is numeric:

```python
min_v = float(df[filter_col].min())
max_v = float(df[filter_col].max())
selected_range = st.sidebar.slider(
    f"Filter Range for `{filter_col}`:",
    min_value=min_v,
    max_value=max_v,
    value=(min_v, max_v),
)
```

This creates a range slider and filters rows within the chosen bounds.

#### If the selected column is categorical or text:

```python
search_query = st.sidebar.text_input(
    f"Search text in `{filter_col}`:"
)
if search_query:
    filtered_df = filtered_df[
        filtered_df[filter_col]
        .astype(str)
        .str.contains(search_query, case=False, na=False)
    ]
```

This creates a case-insensitive substring search. That is highly useful when users want to find rows such as "Germany", "Electronics", or part of a product name.

---

### Step 5: Group aggregation and chart generation

The project provides a dynamic grouping interface to summarize the data:

```python
target_cat = st.selectbox("Group By (Category):", cat_cols, key="agg_cat")
target_num = st.selectbox("Measure (Numeric):", num_cols, key="agg_num")
operation = st.radio("Summary Function:", ["Sum", "Mean", "Count"], horizontal=True)
```

Then the code generates a pandas aggregation based on the chosen operation:

```python
if operation == "Sum":
    summary_data = (
        filtered_df.groupby(target_cat)[target_num]
        .sum()
        .sort_values(ascending=False)
    )
elif operation == "Mean":
    summary_data = (
        filtered_df.groupby(target_cat)[target_num]
        .mean()
        .sort_values(ascending=False)
    )
else:
    summary_data = (
        filtered_df.groupby(target_cat)[target_num]
        .count()
        .sort_values(ascending=False)
    )
```

Finally, it renders the result as a bar chart:

```python
st.bar_chart(summary_data.head(15))
```

#### Why this is powerful

This is a very practical EDA workflow:

- choose a category column such as product type, region, city, or segment
- choose a numeric measure such as revenue, profit, or quantity
- select whether to sum, average, or count that measure
- instantly see the visual trend

This is exactly the type of workflow many business analysts need when exploring new data.

---

### Step 6: Previewing data and descriptive statistics

The app also displays a row preview and a statistical summary:

```python
st.dataframe(filtered_df.head(100), use_container_width=True)
```

and if numeric columns exist:

```python
st.dataframe(filtered_df[num_cols].describe(), use_container_width=True)
```

This lets the user inspect:

- the first few rows of the dataset
- the distributional shape of numeric data
- important summary metrics like mean, median, min, and max

---

## 5. FastAPI backend flow

The API part is designed around the same core tasks but through HTTP endpoints.

### Route registration

```python
app.include_router(analytics_router, prefix="/api/v1")
```

This mounts the analytics router at a versioned path. In modern APIs, this is a common way to support future versioning.

### Endpoint 1: upload summary

```python
@router.post(
    "/upload-summary",
    response_model=DatasetSummaryResponse,
    status_code=status.HTTP_200_OK,
)
async def upload_file_summary(file: UploadFile = File(...)):
```

This endpoint accepts an uploaded file and validates the extension:

```python
if not (
    file.filename.endswith(".csv")
    or file.filename.endswith(".xlsx")
    or file.filename.endswith(".xls")
):
    raise HTTPException(
        status_code=status.HTTP_400_BAD_REQUEST,
        detail="Only CSV and Excel sheets (.xlsx, .xls) are supported.",
    )
```

Then it calls the service function:

```python
return get_dataset_summary(contents, file.filename)
```

### Endpoint 2: data filtering

```python
@router.post("/filter", status_code=status.HTTP_200_OK)
async def filter_file_data(
    file: UploadFile = File(...),
    column_name: str = Query(..., description="Target column to search within"),
    search_value: str = Query(..., description="Value or substring to match"),
    limit: int = Query(50, ge=1, le=500, description="Max rows returned"),
):
```

This endpoint lets a client filter records by a given column and search substring, returning only matched rows.

---

## 6. Service layer: pandas logic

The real analytics work is in app/services/analyze_data.py.

### Loading data from bytes

```python
def load_dataframe_from_bytes(file_contents: bytes, filename: str) -> pd.DataFrame:
    if filename.endswith(".csv"):
        return pd.read_csv(io.BytesIO(file_contents))
    elif filename.endswith((".xlsx", ".xls")):
        return pd.read_excel(io.BytesIO(file_contents))
    else:
        raise ValueError(
            "Unsupported format. Please upload a .csv, .xlsx, or .xls file."
        )
```

This is important because the API receives file bytes, not paths. The code wraps those bytes in `io.BytesIO`, which is the exact object pandas needs for in-memory CSV/Excel parsing.

### Summary generation

```python
def get_dataset_summary(file_contents: bytes, filename: str) -> Dict[str, Any]:
    df = load_dataframe_from_bytes(file_contents, filename)

    numeric_cols = df.select_dtypes(include=["number"]).columns.tolist()
    categorical_cols = df.select_dtypes(
        include=["object", "category", "string"]
    ).columns.tolist()

    df_clean = df.fillna("")
```

This function outputs a dictionary containing:

- filename
- total rows
- total columns
- numeric columns
- categorical columns
- missing cell count
- descriptive statistics
- sample rows

This is a clean API response shape and matches the schema defined in the Pydantic model.

### Filtering logic

```python
def filter_dataset(
    file_contents: bytes,
    filename: str,
    column_name: str,
    search_value: str,
    limit: int = 50,
) -> Dict[str, Any]:
    df = load_dataframe_from_bytes(file_contents, filename)

    if column_name not in df.columns:
        raise ValueError(
            f"Column '{column_name}' not found. Available columns: {list(df.columns)}"
        )

    matched_df = df[
        df[column_name]
        .astype(str)
        .str.contains(search_value, case=False, na=False)
    ]
```

This logic is intentionally generic: it makes the filter work even if the column is numeric, text, or mixed. Converting to string is a practical and portable way to search across the dataset.

---

## 7. API schema design

The response contract is defined in app/api/v1/schemas.py:

```python
class DatasetSummaryResponse(BaseModel):
    filename: str
    total_rows: int
    total_columns: int
    numeric_columns: List[str]
    categorical_columns: List[str]
    missing_cells_count: int
    descriptive_statistics: Dict[str, Dict[str, Any]]
    sample_data: List[Dict[str, Any]]
```

### Why this matters

This model ensures the API returns a predictable structure. It is especially helpful when the frontend or another service consumes the API programmatically.

Without such validation, responses could be inconsistent, and downstream clients might break if a field changes unexpectedly.

---

## 8. Why the project uses both Streamlit and FastAPI

This repo is a really good example of two layers:

### Streamlit layer
Used for interactive human-facing exploration.

Benefits:

- quick user interaction
- real-time charts
- visual filtering
- easy testing of analytics concepts

### FastAPI layer
Used as a backend service API.

Benefits:

- machine-accessible interface
- structured responses
- clean separation of route logic and data logic
- good foundation for future integrations

### Important observation

The current Streamlit app is not fully calling the FastAPI service. It loads the file directly and performs analytics in the UI code itself.

This means the project is currently a hybrid prototype rather than a fully integrated frontend-backend system.

In a production version, a cleaner architecture would be:

1. Streamlit frontend uploads file
2. frontend calls FastAPI endpoint
3. FastAPI analyzes the dataset and sends back JSON
4. Streamlit renders the JSON result in UI

This would align the architecture more tightly and make the backend reusable by other clients.

---

## 9. Data flow in plain English

Here is the essential flow of the project:

1. User opens the Streamlit dashboard
2. User uploads a CSV or Excel file or uses the default dataset
3. The app reads the dataset into a pandas DataFrame
4. The DataFrame is analyzed for numeric and categorical columns
5. User filters rows by a selected column and range or text search
6. User chooses a grouping column and value metric
7. Pandas groups and aggregates the data
8. The results are displayed as charts and tables
9. The FastAPI backend provides a second path for summary and filtering operations over uploaded files

This is a classic EDA flow: inspect, filter, aggregate, summarize, visualize.

---

## 10. Key technical choices and reasoning

### Use of pandas
Pandas is the central technology because it handles:

- CSV and Excel parsing
- DataFrame manipulation
- type detection
- filtering
- groupby aggregations
- summary statistics

This is the ideal library for tabular analytics tasks.

### Use of Streamlit
Streamlit is chosen because it lets developers build a dashboard very quickly without writing separate JavaScript frontends.

It is excellent for:

- internal dashboards
- data exploration tools
- prototypes
- analyst productivity apps

### Use of FastAPI
FastAPI is used because it is fast, modern, and built for API-first data services.

It provides:

- automatic validation using Pydantic
- easy request parsing
- clear error handling
- a clean interface for client systems

---

## 11. Strengths of this project

- Simple and easy to understand
- Uses real pandas operations rather than abstracted libraries
- Covers both UI and backend patterns
- Works with generic datasets instead of hardcoded columns
- Good for learning how Streamlit + FastAPI + pandas fit together

---

## 12. Limitations and areas for improvement

This project is intentionally simple, but there are some opportunities for enhancement:

- the Streamlit UI and FastAPI backend are not fully integrated
- file configuration in app/core/config.py is empty
- there is no database layer
- there are no tests yet
- there is no authentication or multi-user handling
- there is no robust validation for malformed datasets beyond basic parsing

These are not flaws in a learning project; they are natural next-step improvements for production-grade development.

---

## 13. Suggested next enhancements

If this project were expanded, the best next steps would be:

1. Combine the UI and API using a clean shared service layer
2. Add a real configuration management system
3. Add unit tests for summary and filter functions
4. Handle duplicate rows, date parsing, and null-heavy columns more robustly
5. Add export options like CSV and JSON downloads
6. Add charts beyond basic bar charts, such as histograms and scatter plots
7. Add authentication or user management if this becomes a shared internal tool

---

## 14. How to run this project

### Install dependencies

```bash
uv venv
.venv\Scripts\activate
uv pip install -r requirements.txt
```

### Start the Streamlit app

```bash
uv run streamlit run streamlit_app.py
```

### Start the FastAPI app

```bash
uvicorn app.main:app --reload
```

Then you can visit:

- Streamlit dashboard: local Streamlit app port
- FastAPI docs: http://127.0.0.1:8000/docs

---

## 15. Practical understanding of the code

The project is built around a very important principle:

> keep data loading and analysis generic, not hardcoded to one business dataset.

This is why the code uses:

- `df.select_dtypes()`
- dynamic column selection
- generic filtering across any column
- generic group-by logic

This makes the system reusable across sales data, employee records, inventory tables, or any spreadsheet with rows and columns.

That is one of the strongest architectural ideas in this repository.

---

## 16. Final summary

This project is a compact analytics toolkit that demonstrates how to:

- read uploaded spreadsheets
- understand the dataset structure
- filter the data interactively
- build summary metrics
- aggregate information by category
- present results in a user-friendly interface

The combination of Streamlit and FastAPI gives a strong educational foundation for modern data applications. It is simple enough to understand quickly, but structured enough to show the responsibilities of UI, API, schema validation, and service logic.

In short, the project teaches not just pandas and Streamlit, but also good software separation of concerns.

---

## 17. Quick callout: the main idea behind the architecture

If you remember only one thing from this walkthrough, remember this:

- Streamlit is the front-end analyst experience
- FastAPI is the backend service layer
- pandas is the computational engine
- the app/services layer is where reusable analytics logic lives

This separation is what keeps the application understandable and extensible.
