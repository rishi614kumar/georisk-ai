# GeoRisk AI
An Agentic Geospatial AI Chatbot for Autonomous Multi-Domain Urban Risk Analysis in NYC.

Further details are available in our [Final Report](https://github.com/rishi614kumar/georisk-ai/blob/main/Team17%20Final%20Report.pdf).

## Setup Instructions

### Prerequisites
1. **Python**: Ensure Python 3.8+ is installed on your system.
2. **Dependencies**: Install the required Python packages using `pip`.
3. **Chainlit**: Install Chainlit globally using pip if not already installed:
   ```bash
   pip install chainlit
   ```

### Environment Configuration
1. Create a `.env` file in the root directory of the project (if it doesn't already exist).
2. Add the following environment variables to the `.env` file:

   ```env
   GEOCLIENT_API_KEY=<your_geoclient_api_key>
   MAPPLUTO_GDB_PATH=<path_to_mappluto_gdb>
   LION_GDB_PATH=<path_to_lion_gdb>
   NTA_PATH=https://data.cityofnewyork.us/resource/9nt8-h7nd.geojson
   GEMINI_API_KEY=<your_gemini_api_key>
   CRIME_PATH=<local path to downloaded crime data file>
   CHAINLIT_AUTH_SECRET=<a secure token or random string>
   CHAINLIT_ADMIN_USER=<admin_username>
   CHAINLIT_ADMIN_PASSWORD=<admin_password>
   CHAINLIT_DB_URL=sqlite+aiosqlite:///./chainlit_history.db
   ```

   The crime data XLS file is already included in the modified_dataset folder. Sourced from: [NYPD Historical Crime Data – Seven Major Felony Offenses by Precinct (2000–2024)](https://www.nyc.gov/assets/nypd/downloads/excel/analysis_and_planning/historical-crime-data/seven-major-felony-offenses-by-precinct-2000-2024.xls)

   Replace the placeholders (e.g., `<your_geoclient_api_key>`) with the actual values for your environment.

   You can generate a secure value for `CHAINLIT_AUTH_SECRET` with:

   ```bash
   chainlit create-secret
   ```

   Copy the generated secret into your `.env` file.

### Installation
1. Clone the repository:
   ```bash
   git clone https://github.com/rishi614kumar/georisk-ai.git
   ```
2. Navigate to the project directory:
   ```bash
   cd georisk-ai
   ```
3. Install the required dependencies:
   ```bash
   pip install -r requirements.txt
   ```

### Running the Application
Run the application using Chainlit:
```bash
chainlit run app.py -w
```

This will start the chatbot application, and you can interact with it through the Chainlit interface.

## Code File Instructions
### adapters
- **coord.py**  
coord.py: This module provides utility functions for converting between longitude/latitude and NYC tax lot identifiers (BBL).
- **epsg.py**  
  Centralizes coordinate reference system (CRS) transformations between geographic and projected coordinates.
- **geometry.py**  
Convert point-based datasets into projected geometries and intersect them with buffered tax-lot polygons for location-aware data selection.
- **nta.py**  
  Convert between BBL and NTA.
- **precinct.py**
   Convert between BBL and Precinct.
- **schemas.py**  
  A unified geospatial data schema used to aggregate identifiers, location attributes, and source provenance across multiple adapters.
- **street_span.py**  
  Translate linear street segments or intersection spans into BBL.
- **surrounding.py**
   Obtain surrounding bbls from one target bbl.
### api
- **GeoClient.py**  
  Provides a fault-tolerant wrapper around the NYC Geoclient API, standardizing responses and bridging API lookups with local spatial fallbacks when necessary.
- **socrata_client.py**  
  Encapsulates Socrata API access with retry and throttling handling to ensure stable ingestion of NYC Open Data datasets.
### config
- **settings.py**  
  Defines environment variables, dataset registries, spatial buffer defaults, and prompt templates that guide data retrieval, risk categorization, and location parsing.
### data
- **lion.py**  
  Provides cached access to the LION and LION14 street network datasets.
- **nta2020.py**  
  Loads and caches the NYC 2020 Neighborhood Tabulation Area (NTA) data.

- **pluto.py**  
  Acts as the parcel-level backbone of the system, reading MapPLUTO once from disk and exposing multiple lightweight, cached views keyed by BBL.

### llm
- **LLMInterface.py**  
  Defines a modular LLM chat interface with pluggable backends, providing a unified abstraction for interacting with different language model providers and handling retries, safety filtering, and conversation state.

- **LLMParser.py**  
  Implements the LLM-based query router that classifies user intent, extracts NYC locations, and selects relevant datasets using structured prompts, few-shot examples, and robust fallbacks.

### prompts
- **app_prompts.py**  
  Centralizes all system prompts, meta-prompts, and decision prompts used by the application to control LLM behavior across conversational flow, dataset loading, risk summarization, and follow-up generation.
### scripts
- **ConversationalAgent.py**  
  Orchestrates the end-to-end conversational workflow by coordinating LLM decisions, geospatial resolution, dataset filtering, risk summarization, and follow-up generation through modular decision units.
- **ConversationalUnit.py**
   This module defines the **unit-based execution framework** for the conversational agent. Each unit encapsulates a single decision or processing step in the end-to-end LLM-driven workflow, enabling modular, traceable, and composable dialogue reasoning.
- **DataHandler.py**  
  This module defines the **dataset abstraction and orchestration layer** for the NYC risk analysis system. 

- **GeoBundle.py**  
  This module provides the **geographic resolution and normalization layer** that converts addresses or BBLs into a unified `GeoBundle` representation. 
- **GeoScope.py**  
  This module implements the **geospatial resolution and filtering engine** that converts parsed addresses into resolved BBLs, street spans, and surrounding spatial units, and then constructs dataset-specific spatial query filters.
- **RiskSummarizer.py**  
  This module implements the **risk summarization layer** of the GeoRisk AI pipeline.

### test
-  This folder contains **unit and integration tests for the geospatial adapter layer**, primarily validating that different geographic units (e.g., BBL, longitude/latitude, precinct, surrounding areas) are resolved and converted correctly.  

### modified_dataset

This folder contains **locally modified datasets** that do not have a public API
or are unsuitable for direct Socrata querying.

These datasets are preprocessed to:
- Normalize column names
- Align geographic units (e.g., precinct, borough)
- Enable efficient filtering inside GeoRisk AI

#### Current Contents

- `felony_by_precinct_cleaned.csv`  
  A cleaned and standardized version of felony crime data aggregated by NYPD precinct.
  This version is optimized for:
  - Fast local filtering
  - Consistent precinct identifiers
  - Compatibility with the `PRECINCT` geo unit

Additional datasets without APIs should be placed in this folder after similar preprocessing.
## Instructions for Adding More Datasets

This project is designed to be **dataset-extensible**. Adding a new dataset requires **three coordinated steps** so that the system can (1) retrieve the data, (2) understand its meaning, and (3) apply correct geographic filtering.

---

### 1. Register the Dataset Source (API or Flat File)

Add the dataset’s **data source** to `settings.py`.

#### Socrata API datasets
If the dataset is hosted on NYC OpenData (Socrata):

- Add the dataset name and its **4×4 Socrata API ID** to `DATASET_API_IDS`
- If the dataset uses a non-default domain, add an override to `SOCRATA_DOMAIN_OVERRIDES`

```python
DATASET_API_IDS = {
    "My New Dataset": "abcd-1234",
}

SOCRATA_DOMAIN_OVERRIDES = {
    "My New Dataset": "data.cityofnewyork.us",
}
```
If it is Flat file datasets (csv/xlsx)
- If the dataset is stored locally:
Add its file path to FLATFILE_PATHS
```python
FLATFILE_PATHS = {
    "My New Dataset": os.getenv("MY_NEW_DATASET_PATH"),
}
```

### 2. Add a Human-Readable Dataset Description

Each dataset must have a clear semantic description so the LLM can reason about relevance and risk.

Add an entry to DATASET_DESCRIPTIONS in settings.py:
```python
DATASET_DESCRIPTIONS = {
    "My New Dataset": (
        "Brief explanation of what the dataset contains, "
        "what kind of risk or context it represents, "
        "and why it may be relevant to site analysis."
    ),
}
```
### 3. Configure Geographic Filtering Behavior (GeoConfig)

Each dataset must declare how it should be spatially filtered.
This is done in the DATASET_CONFIG block in settings.py.

You must specify:
- geo_unit: what geographic identifier the dataset uses
- surrounding: whether nearby areas should be included

Example configurations:
```python
DATASET_CONFIG = {
    "My New Dataset": {
        "geo_unit": "BBL",        # BBL, BBL_SPLIT, PRECINCT, NTA, LONLAT, STREETSPAN, BOROUGH
        "mode": "street",         # street-based or radius-based filtering
        "surrounding": True,      # include nearby units or not
    }
}
```
For datasets that split BBL fields or use custom column names:
```python

"My New Dataset": {
    "geo_unit": "BBL_SPLIT",
    "Borough": "M",
    "Borough_form": 1,
    "cols": {"block": "propertyblock", "lot": "propertylot"},
    "col_names": {"borough": "propertyborough"},
    "mode": "street",
    "surrounding": True,
}
```
For point-based datasets using geometry columns:
```python
"My New Dataset": {
    "geo_unit": "LONLAT",
    "col_names": {"geometry": "the_geom"},
    "mode": "radius",
    "surrounding": False,
}
```
### 4. Map the Dataset to Risk Categories

Finally, associate the dataset with one or more risk categories so it can be selected during query routing.

Add it to cat_to_ds:
```python
cat_to_ds = {
    "Environmental & Health Risks": [
        "My New Dataset",
    ],
}
```
This ensures the dataset is:
- Discoverable by the LLM
- Used only when semantically relevant
- Combined correctly with other datasets in multi-category queries

### Summary
To add a dataset, make sure all four are complete:
- Data source registered (DATASET_API_IDS or FLATFILE_PATHS)
- Human-readable description added (DATASET_DESCRIPTIONS)
- Geographic behavior defined (DATASET_CONFIG)
- Category mapping updated (cat_to_ds)


## Known Issues

During our evaluation and after our code freeze, we identified the following issues:

- **Initial Interaction Requirement**  
  Users must wait for the first response from **GeoRisk AI** before continuing the conversation.  
  The system requires an initialized conversational context to function correctly.

- **Multiple Address Handling**  
  When a user inputs **two or more addresses**, `GeoScope` sometimes applies
  spatial filtering **only to the first resolved address**.

- **Ambiguous or Incomplete Address Parsing**  
  If an address:
  - does not include a house number **and**
  - is not a clear intersection  
  the address parser may fail or incorrectly resolve the location  
  (e.g., `LGA` → *LaGuardia Road, East Elmhurst, Queens*).

- **Radius Filtering Limitation**  
  The `within_circle` spatial filter currently uses a **static radius of 100 meters**.  
  User-specified radius values are **not yet supported**.

  - **NTA Data Inconsistency**  
  Conversions from BBL to NTA are available, however, the converted NTA values do not always line up with the NTA values provided in the Population by NTA dataset.

- **GeoClient Address Range Errors**  
  Some valid NYC addresses (e.g.,  
  *100 Latourette Ln, Staten Island, NY 10314*)  
  may return `ERROR: ADDRESS NUMBER OUT OF RANGE` from the GeoClient API,  
  even though a valid BBL exists for the location.

- **Ambiguous Street Intersections**  
  If two streets intersect **more than once** (e.g., *Canal St & Bowery*),  
  users must specify a direction or compass qualifier.  
  This disambiguation logic is **not yet implemented**.
