# NYC-Capital-Projects
Public Building vs. Infrastructure Classification
NYC Capital Projects — Public Building vs. Infrastructure Classification
Classification hierarchy
Rule Spreadsheet → Sponsor Agency → Managing Agency Fallback → Project Text → Strong Phrases → Keyword Scores → FMS ID/PID Consistency → Manual Review → Final Dataset

Each numbered section below contains:

What the step does
Why the step is needed
What result to expect

Python Setup
The notebook uses:

Pandas for reading, transforming, classifying, and saving tabular data.
re for regular-expression text matching.
Path from pathlib for cleaner and safer file-path handling.

import pandas as pd
import re
from pathlib import Path

### Step 1: File Paths

**What this step does**

This section defines the locations of all files used by the classification workflow.

- `input_file` points to the already merged NYC Capital Projects dataset.
- `rule_file` points to the Excel spreadsheet containing the PB/Infrastructure classification rules.
- `output_folder` defines where the results will be saved.
- `output_file` is the final classified dataset.
- `review_file` stores projects that still cannot be classified automatically.
- `mkdir(..., exist_ok=True)` creates the output folder if it does not already exist.

**Why this step is needed**

Keeping file locations in one place makes the notebook easier to update and prevents paths from being repeated throughout the code.

**Expected result**

input_file = Path(
    r"C:\Users\Dell User\NYC_Capital_Projects_Wide_Format.csv"
)

rule_file = Path(
    r"C:\Users\Dell User\Desktop\Fordham course matrials\Applied Project"
    r"\Revised excel spreadsheet(PB & I).xlsx"
)

output_folder = Path(
    r"C:\Users\Dell User\Desktop\capital_projects dataset"
)

output_file = output_folder / "NYC_Capital_Projects_PB_I_Classified.csv"
review_file = output_folder / "Projects_Needing_Review.csv"

output_folder.mkdir(parents=True, exist_ok=True)


### Step 2:Load Merged Dataset + Rule File

**What this step does**

The merged capital-project dataset is loaded into `df`, while the Excel PB/I rule spreadsheet is loaded into `rules`.

`low_memory=False` is used when reading the CSV so Pandas processes mixed-type columns more consistently and avoids the common mixed-type warning.

The code also prints the dimensions of both datasets. If the merged dataset contains more than 100,000 rows, it prints a warning because the earlier many-to-many merge produced an unusually large dataset.

**Why this step is needed**

All later classification logic depends on these two inputs:

1. the project-level merged data; and
2. the agency-level PB/I rule table.

df = pd.read_csv(
    input_file,
    low_memory=False
)

rules = pd.read_excel(
    rule_file
)

print("Merged dataset shape:", df.shape)
print("Rule file shape:", rules.shape)

if len(df) > 100000:
    print(
        "\nWARNING:"
        f"\nThe merged dataset contains {len(df):,} rows."
        "\nClassification will continue, but this may be the"
        "\nolder many-to-many merged dataset with duplicated rows."
    )


### Step 3:Check Required Columns

**What this step does**

The code verifies that the merged dataset contains every field required for classification:

- `Sponsor Agency`
- `Managing Agency_schedule`
- `Agency Project Description`
- `Agency Project Name`
- `FMS Project Name`
- `Ten Year Plan Category`
- `FMS ID`
- `PID`

If any required field is missing, the notebook stops with a clear `ValueError`.

**Why this step is needed**

It prevents the classification pipeline from continuing with an incomplete or differently structured dataset. Without this validation, later code could fail with harder-to-understand errors.

**Expected result**

If all required columns exist, nothing is printed and the notebook continues. If any are missing, their names are reported.

required_columns = [
    "Sponsor Agency",
    "Managing Agency_schedule",
    "Agency Project Description",
    "Agency Project Name",
    "FMS Project Name",
    "Ten Year Plan Category",
    "FMS ID",
    "PID"
]

missing_columns = [
    col for col in required_columns
    if col not in df.columns
]

if missing_columns:
    raise ValueError(
        f"Missing required columns: {missing_columns}"
    )

### Step 4: Clean Text Functions

**What this step does**

Two reusable cleaning functions are created:

- `clean(series)` cleans an entire Pandas column.
- `clean_value(value)` cleans a single text value.

The cleaning process:

- replaces missing values with blank text,
- converts values to lowercase,
- standardizes apostrophes and dashes,
- removes repeated spaces,
- trims spaces at the beginning and end.

**Why this step is needed**

Agency and project names may appear with different capitalization, punctuation, or spacing. Standardizing the text allows reliable rule matching and keyword searching.

**Expected result**

Two functions are created. No classification is performed yet.

def clean(series):
    return (
        series
        .fillna("")
        .astype(str)
        .str.lower()
        .str.replace("’", "'", regex=False)
        .str.replace("–", "-", regex=False)
        .str.replace(r"\s+", " ", regex=True)
        .str.strip()
    )


def clean_value(value):
    if pd.isna(value):
        return ""

    value = str(value).lower().strip()
    value = value.replace("’", "'")
    value = value.replace("–", "-")
    value = re.sub(r"\s+", " ", value)

    return value

### Step 5: Prepare Project Text

**What this step does**

The main text fields used in the classification are cleaned:

- Sponsor Agency
- Managing Agency
- Agency Project Description
- Agency Project Name
- FMS Project Name
- Ten Year Plan Category

The project-related fields are then combined into one searchable text string called `project_text`.

The conceptual priority is:

**Project Description → Agency Project Name → FMS Project Name → Ten Year Plan Category**

**Why this step is needed**

Some projects have incomplete descriptions or missing sponsor information. Combining several descriptive fields gives the classification rules more information to work with.

**Expected result**

Cleaned text variables and a combined `project_text` series are available for later keyword analysis.

sponsor = clean(df["Sponsor Agency"])
managing = clean(df["Managing Agency_schedule"])

description = clean(
    df["Agency Project Description"]
)

project_name = clean(
    df["Agency Project Name"]
)

fms_name = clean(
    df["FMS Project Name"]
)

category = clean(
    df["Ten Year Plan Category"]
)

# Priority:
# Description -> Project Name -> FMS Name -> Category

project_text = (
    description + " " +
    project_name + " " +
    fms_name + " " +
    category
)

### Step 6: Read PB/I Rules from Excel

**What this step does**

This step converts the Excel reference rules into three Python agency groups:

- `PB_AGENCIES` — agencies whose rule says **Assume PB**
- `I_AGENCIES` — agencies whose rule says **Assume Infra**
- `VARIABLE_AGENCIES` — agencies whose classification depends on the individual project

The code also creates `SPECIAL_ALIASES` so formal agency names can match abbreviations used in the NYC dataset, such as `DEP`, `DOT`, `DPR`, `NYPD`, `NYPL`, and `H+H`.

Parenthetical text is removed from agency names, aliases are added, and duplicate names are removed.

**Why this step is needed**

The Excel spreadsheet is the primary classification authority. This step translates its human-readable rules into machine-readable lists while accounting for agency abbreviations.

**Expected result**

The notebook prints the fixed PB agencies, fixed Infrastructure agencies, and project-dependent agencies extracted from the rule file.

rules.columns = (
    rules.columns
    .astype(str)
    .str.strip()
)

print("\nRule file columns:")
print(rules.columns.tolist())


# Abbreviations / names that may appear in NYC dataset
SPECIAL_ALIASES = {

    "agency for children's services": ["acs"],
    "brooklyn public library": ["bpl"],
    "city council": ["city council"],

    "district attorney (brooklyn)": [
        "da brooklyn"
    ],

    "district attorney (queens)": [
        "da queens"
    ],

    "department of citywide administrative services": [
        "dcas"
    ],

    "department of cultural affairs": [
        "dcla"
    ],

    "department of city planning": [
        "dcp"
    ],

    "department of design and construction": [
        "ddc"
    ],

    "department of environmental protection": [
        "dep"
    ],

    "department for the aging": [
        "dfta"
    ],

    "department of homeless services": [
        "dhs"
    ],

    "department of correction": [
        "doc"
    ],

    "department of health and mental hygiene": [
        "dohmh"
    ],

    "department of transportation": [
        "dot"
    ],

    "department of parks and recreation": [
        "dpr",
        "parks"
    ],

    "department of sanitation": [
        "dsny"
    ],

    "economic development corporation": [
        "edc"
    ],

    "fire department of new york": [
        "fdny"
    ],

    "nyc health + hospitals": [
        "h+h",
        "hhc"
    ],

    "housing preservation & development": [
        "hpd"
    ],

    "human resources administration": [
        "hra"
    ],

    "nyc law department": [
        "law"
    ],

    "mayor's office of contract services": [
        "mocs"
    ],

    "mayor's office of resiliency": [
        "mor"
    ],

    "nyc housing authority": [
        "nycha"
    ],

    "new york police department": [
        "nypd"
    ],

    "new york public library": [
        "nypl"
    ],

    "queens public library": [
        "qpl"
    ]
}


PB_AGENCIES = []
I_AGENCIES = []
VARIABLE_AGENCIES = []


for _, row in rules.iterrows():

    # Clean sponsor agency
    agency = clean_value(
        row.get("Sponsor Agency", "")
    )

    # Clean PB / Infrastructure classification
    classification = clean_value(
        row.get("PB or Infra", "")
    )

    # Remove text inside parentheses from agency name
    base_agency = re.sub(
        r"\s*\([^)]*\)",
        "",
        agency
    ).strip()

    # Build aliases
    aliases = []

    if base_agency:
        aliases.append(base_agency)

    # Add special abbreviations / aliases
    for key, values in SPECIAL_ALIASES.items():
        if key in agency:
            aliases.extend(values)

    # Remove duplicate aliases
    aliases = list(set(aliases))

   
    # CLASSIFICATION
    

    if "assume pb" in classification:

        PB_AGENCIES.extend(
            aliases
        )

    elif "assume infra" in classification:

        I_AGENCIES.extend(
            aliases
        )

    elif (
        "pb or i" in classification
        or "depends on project" in classification
    ):

        VARIABLE_AGENCIES.extend(
            aliases
        )


# Remove duplicates and sort
PB_AGENCIES = sorted(set(PB_AGENCIES))
I_AGENCIES = sorted(set(I_AGENCIES))
VARIABLE_AGENCIES = sorted(set(VARIABLE_AGENCIES))


print("\nFixed PB agencies:")
print(PB_AGENCIES)

print("\nFixed Infrastructure agencies:")
print(I_AGENCIES)

print("\nProject-dependent agencies:")
print(VARIABLE_AGENCIES)


### Step 7: Build Safe Regular-Expression Patterns

**What this step does**

The `make_pattern()` function converts each agency list into one regular-expression search pattern.

The function:

- removes blank entries,
- removes duplicates,
- sorts longer names before shorter names,
- escapes special characters,
- uses word boundaries to reduce accidental partial matches.

Three patterns are created:

- `pb_agency_pattern`
- `i_agency_pattern`
- `variable_pattern`

**Why this step is needed**

Agency names can appear inside longer text strings. Regex patterns allow efficient vectorized matching across thousands of rows without looping through each row individually.

**Expected result**

Three reusable patterns are created for the classification steps that follow.

def make_pattern(words):

    words = [
        word for word in words
        if word
    ]

    if not words:
        return r"(?!x)x"

    words = sorted(
        set(words),
        key=len,
        reverse=True
    )

    return (
        r"(?<!\w)(?:"
        +
        "|".join(
            re.escape(word)
            for word in words
        )
        +
        r")(?!\w)"
    )


pb_agency_pattern = make_pattern(
    PB_AGENCIES
)

i_agency_pattern = make_pattern(
    I_AGENCIES
)

variable_pattern = make_pattern(
    VARIABLE_AGENCIES
)

### Step 8: Create the Classification Column

**What this step does**

A new column named `PB_or_I` is added to the merged dataset.

Every row initially receives a missing value (`pd.NA`).

**Why this step is needed**

This column becomes the final classification field. Each later rule fills it with either:

- `PB` = Public Building
- `I` = Infrastructure

Starting with missing values also makes it easy to identify which projects still need further analysis.

**Expected result**

The dataset now contains a new `PB_or_I` column, initially blank for all rows.

df["PB_or_I"] = pd.NA

### Step 9: Sponsor Agency Classification

**What this step does**

Sponsor Agency is used as the first classification layer.

The code first identifies sponsors that belong to the project-dependent group. These agencies are deliberately excluded from automatic agency-only classification.

For all other sponsors:

- matches to the fixed PB list receive `PB`;
- matches to the fixed Infrastructure list receive `I`.

The code then prints the number of PB, I, and still-unclassified rows.

**Why this step is needed**

Sponsor Agency is the strongest agency-level indicator in the rule spreadsheet. Applying it first allows a large share of projects to be classified directly from the reference rules.

**Expected result**

Many rows receive their first PB/I classification, while variable-agency and unmatched projects remain blank.


# Project-dependent agencies should not be
# automatically classified from agency name.

variable_sponsor = sponsor.str.contains(
    variable_pattern,
    regex=True
)


# Fixed Public Building
df.loc[
    ~variable_sponsor
    &
    sponsor.str.contains(
        pb_agency_pattern,
        regex=True
    ),
    "PB_or_I"
] = "PB"


# Fixed Infrastructure
df.loc[
    ~variable_sponsor
    &
    sponsor.str.contains(
        i_agency_pattern,
        regex=True
    ),
    "PB_or_I"
] = "I"


print("\nAfter Sponsor Agency rules:")

print(
    df["PB_or_I"]
    .value_counts(dropna=False)
)

### Step 10: Managing Agency Fallback

**What this step does**

Managing Agency is used only when Sponsor Agency is blank and the project is still unclassified.

The code excludes DDC and EDC from automatic fallback classification because those organizations may manage both Public Building and Infrastructure work.

For other managing agencies:

- matches to the PB agency list receive `PB`;
- matches to the Infrastructure agency list receive `I`.

**Why this step is needed**

Some projects are missing Sponsor Agency information. Managing Agency provides a secondary source without overriding stronger sponsor-based evidence.

**Expected result**

Additional projects with missing sponsor information may receive a PB/I classification.

# Only use Managing Agency when Sponsor Agency is blank.

sponsor_missing = sponsor.eq("")

# DDC and EDC may manage both PB and Infrastructure.
unsafe_manager = managing.str.contains(
    r"\bddc\b|\bedc\b|design and construction",
    regex=True
)


df.loc[
    df["PB_or_I"].isna()
    &
    sponsor_missing
    &
    ~unsafe_manager
    &
    managing.str.contains(
        pb_agency_pattern,
        regex=True
    ),
    "PB_or_I"
] = "PB"


df.loc[
    df["PB_or_I"].isna()
    &
    sponsor_missing
    &
    ~unsafe_manager
    &
    managing.str.contains(
        i_agency_pattern,
        regex=True
    ),
    "PB_or_I"
] = "I"

### Step 11: Define Infrastructure Keywords

**What this step does**

A vocabulary of terms associated with Infrastructure work is created.

The list covers several categories, including:

- roads and transportation,
- bridges,
- water systems,
- sewer and drainage,
- utilities,
- traffic systems,
- coastal and waterfront work,
- wastewater and environmental systems,
- parks and site improvements.

Examples include `roadway`, `bridge`, `water main`, `sewer`, `stormwater`, `utility relocation`, and `street lighting`.

**Why this step is needed**

Project-dependent agencies cannot always be classified from agency name alone. Infrastructure keywords provide project-level evidence from descriptions and project names.

**Expected result**

`INFRA_KEYWORDS` becomes available for later scoring.

INFRA_KEYWORDS = [

    # Roads / Transportation
    "road",
    "roadway",
    "street",
    "streetscape",
    "intersection",
    "sidewalk",
    "curb",
    "pavement",
    "paving",
    "pedestrian",
    "bike lane",
    "bicycle lane",
    "highway",

    # Bridges
    "bridge",
    "overpass",
    "underpass",
    "viaduct",

    # Water
    "water main",
    "watermain",
    "water supply",
    "water distribution",

    # Sewer / Drainage
    "sewer",
    "storm sewer",
    "sanitary sewer",
    "combined sewer",
    "drainage",
    "stormwater",
    "catch basin",

    # Utilities
    "utility",
    "utilities",
    "utility relocation",

    # Transportation Systems
    "traffic signal",
    "street lighting",

    # Coastal / Waterfront
    "bulkhead",
    "shoreline",
    "coastal",
    "waterfront",
    "pier",
    "dock",
    "esplanade",

    # Environmental
    "wastewater",
    "treatment plant",
    "pump station",
    "pumping station",
    "reservoir",
    "aqueduct",

    # Parks / Site Work
    "playground",
    "athletic field",
    "synthetic turf",
    "greenway",
    "plaza",
    "park reconstruction",
    "park improvement",

    # General Infrastructure
    "site infrastructure",
    "infrastructure improvement",
    "infrastructure improvements"
]

### Step 12: Define Public Building Keywords

**What this step does**

A vocabulary of terms associated with Public Building projects is created.

The list includes:

- buildings and facilities,
- libraries,
- police and fire facilities,
- correctional facilities,
- hospitals and health centers,
- museums and cultural centers,
- community and recreation centers,
- courthouses,
- sanitation garages,
- comfort stations,
- building-system work such as roofs, HVAC, elevators, and interior renovation.

**Why this step is needed**

These keywords provide evidence that a project is primarily a building or facility project rather than horizontal/civil infrastructure.

**Expected result**

`PB_KEYWORDS` becomes available for later scoring.

PB_KEYWORDS = [

    "building",
    "facility",
    "facilities",

    "library",

    "police precinct",
    "precinct",

    "firehouse",
    "fire station",

    "correctional facility",
    "jail",
    "detention",

    "hospital",
    "health center",
    "clinic",
    "medical center",

    "museum",
    "cultural center",

    "community center",
    "recreation center",

    "office building",
    "administrative building",

    "courthouse",

    "sanitation garage",
    "garage facility",

    "comfort station",
    "field house",
    "fieldhouse",

    "roof replacement",
    "roof renovation",

    "hvac",
    "boiler",
    "elevator",

    "facade",
    "building envelope",

    "interior renovation",
    "interior construction"
]

### Step 13: Define Strong Project Phrases

**What this step does**

Two lists of highly specific phrases are created:

- `STRONG_INFRA`
- `STRONG_PB`

Strong Infrastructure phrases include items such as `roadway reconstruction`, `water main replacement`, `sanitary sewer`, and `utility relocation`.

Strong Public Building phrases include items such as `library renovation`, `police precinct`, `hospital renovation`, and `building rehabilitation`.

Each list is converted into a regex pattern.

**Why this step is needed**

Some phrases are much more informative than generic words such as `facility` or `street`. Strong rules are therefore applied before general keyword scoring.

**Expected result**

Two high-confidence text patterns are ready for use in the next step.

STRONG_INFRA = [

    "water main",
    "sanitary sewer",
    "storm sewer",
    "combined sewer",

    "roadway reconstruction",
    "street reconstruction",

    "bridge reconstruction",

    "sewer reconstruction",

    "water main replacement",

    "street lighting",

    "utility relocation",

    "drainage improvements",

    "stormwater",

    "bulkhead reconstruction",

    "coastal resiliency",

    "site infrastructure",

    "infrastructure improvements"
]


STRONG_PB = [

    "library renovation",
    "library reconstruction",

    "police precinct",

    "firehouse",
    "fire station",

    "hospital renovation",
    "hospital building",

    "health center",

    "community center",
    "recreation center",

    "sanitation garage",

    "correctional facility",

    "courthouse",

    "museum renovation",

    "building renovation",
    "building rehabilitation"
]


strong_infra_pattern = "|".join(
    re.escape(x)
    for x in STRONG_INFRA
)

strong_pb_pattern = "|".join(
    re.escape(x)
    for x in STRONG_PB
)


### Step 14:Apply Strong Rules to Unclassified Projects

**What this step does**

Strong project phrases are searched only among rows whose `PB_or_I` value is still missing.

- Strong Infrastructure matches are assigned `I`.
- Strong Public Building matches are assigned `PB`.

Existing agency-based classifications are not overwritten.

**Why this step is needed**

This preserves the hierarchy of evidence: agency rules remain authoritative when available, while strong project-specific evidence is used to resolve remaining cases.

**Expected result**

A further group of previously unresolved projects receives a classification.

df.loc[
    df["PB_or_I"].isna()
    &
    project_text.str.contains(
        strong_infra_pattern,
        regex=True
    ),
    "PB_or_I"
] = "I"


df.loc[
    df["PB_or_I"].isna()
    &
    project_text.str.contains(
        strong_pb_pattern,
        regex=True
    ),
    "PB_or_I"
] = "PB"

### Step 15: Calculate Keyword Scores

**What this step does**

For every project, the code counts how many Infrastructure keywords and how many Public Building keywords appear in `project_text`.

This creates:

- `infra_score`
- `pb_score`

Each matching keyword adds one point to the corresponding score.

**Why this step is needed**

Many project descriptions contain several types of terms. A scoring approach uses the overall balance of evidence instead of relying on one generic word.

**Expected result**

Each row receives two internal scores that are used in the next step.

infra_score = sum(

    project_text.str.contains(
        re.escape(word),
        regex=True
    ).astype(int)

    for word in INFRA_KEYWORDS
)


pb_score = sum(

    project_text.str.contains(
        re.escape(word),
        regex=True
    ).astype(int)

    for word in PB_KEYWORDS
)

### Step 16: Classify Remaining Projects by Score

**What this step does**

Only projects that are still unclassified are evaluated.

- If `infra_score` is greater than `pb_score` and at least one Infrastructure keyword was found, the project is classified `I`.
- If `pb_score` is greater than `infra_score` and at least one PB keyword was found, the project is classified `PB`.
- Ties and projects with no keyword evidence remain unresolved.

**Why this step is needed**

This resolves ambiguous projects using the relative strength of their textual evidence while avoiding forced classifications when evidence is weak or tied.

**Expected result**

More previously blank `PB_or_I` values are filled.

remaining_mask = (
    df["PB_or_I"].isna()
)


df.loc[
    remaining_mask
    &
    (infra_score > pb_score)
    &
    (infra_score > 0),
    "PB_or_I"
] = "I"


df.loc[
    df["PB_or_I"].isna()
    &
    (pb_score > infra_score)
    &
    (pb_score > 0),
    "PB_or_I"
] = "PB"

### Step 17: Use Consistent FMS ID Classification

**What this step does**

Projects may appear repeatedly across reporting periods under the same `FMS ID`.

The code groups already-classified rows by `FMS ID`. If every classified occurrence of an FMS ID has the same PB/I value, that classification is treated as consistent and can fill blank rows with the same FMS ID.

If an FMS ID has conflicting PB and I classifications, no value is propagated.

**Why this step is needed**

This improves consistency across repeated records without forcing a classification when historical rows disagree.

**Expected result**

Some unclassified repeated records are filled using reliable project-level consistency.

fms_classification = (
    df[
        df["PB_or_I"].notna()
        &
        df["FMS ID"].notna()
    ]
    .groupby(
        "FMS ID"
    )["PB_or_I"]
    .agg(
        lambda x:
        x.iloc[0]
        if x.nunique() == 1
        else pd.NA
    )
)


fms_fill = (
    df["FMS ID"]
    .map(
        fms_classification
    )
)


df.loc[
    df["PB_or_I"].isna()
    &
    fms_fill.notna(),
    "PB_or_I"
] = fms_fill

### Step 18: Use Consistent PID Classification

**What this step does**

The same consistency logic is repeated using `PID`.

If all classified records sharing a PID have one consistent category, that PB/I value is used to fill still-missing rows with the same PID.

Conflicting PID classifications are left unresolved.

**Why this step is needed**

PID provides another project identifier and can resolve records that were not filled through FMS ID.

**Expected result**

Additional repeated project records may receive a consistent PB/I classification.

pid_classification = (
    df[
        df["PB_or_I"].notna()
        &
        df["PID"].notna()
    ]
    .groupby(
        "PID"
    )["PB_or_I"]
    .agg(
        lambda x:
        x.iloc[0]
        if x.nunique() == 1
        else pd.NA
    )
)


pid_fill = (
    df["PID"]
    .map(
        pid_classification
    )
)


df.loc[
    df["PB_or_I"].isna()
    &
    pid_fill.notna(),
    "PB_or_I"
] = pid_fill


### Step 19: Review Final Classification Results

**What this step does**

The notebook prints the count of:

- `PB`
- `I`
- missing/unclassified values

It also calculates the total number of unresolved rows.

**Why this step is needed**

This is the first overall quality-control checkpoint after all automated classification rules have been applied.

**Expected result**

A classification summary is displayed, including the number of projects that still require review.

print("FINAL CLASSIFICATION RESULTS")


print(
    df["PB_or_I"]
    .value_counts(
        dropna=False
    )
)


remaining = (
    df["PB_or_I"]
    .isna()
    .sum()
)

print(
    "\nUnclassified rows:",
    remaining
)

### Step 20: Save Projects Needing Web/Manual Review

**What this step does**

If unresolved projects remain, the notebook creates a smaller review dataset containing the most useful fields:

- FMS ID
- PID
- Sponsor Agency
- Managing Agency
- Agency Project Description
- Agency Project Name
- FMS Project Name
- Ten Year Plan Category

Duplicate review rows are removed and the file is saved as `Projects_Needing_Review.csv`.

**Why this step is needed**

Ambiguous projects should not be assigned arbitrarily. This file isolates them for targeted research using official NYC sources or other project documentation.

**Expected result**

A review CSV is created only when unresolved projects exist.

if remaining > 0:

    unresolved = df.loc[
        df["PB_or_I"].isna(),
        [
            "FMS ID",
            "PID",
            "Sponsor Agency",
            "Managing Agency_schedule",
            "Agency Project Description",
            "Agency Project Name",
            "FMS Project Name",
            "Ten Year Plan Category"
        ]
    ].drop_duplicates()


    unresolved.to_csv(
        review_file,
        index=False
    )


    print(
        "\nProjects requiring manual/web review saved to:"
    )

    print(
        review_file
    )

### Step 21: Remove Old Unwanted Classification Columns

**What this step does**

If older versions of the dataset contain the following audit fields, they are removed:

- `Classification_Method`
- `Classification_Basis`
- `Classification_Confidence`
- `Classification_Source`

`errors="ignore"` allows the code to continue even when these columns are not present.

**Why this step is needed**

The requested final dataset should contain only the main PB/I result column rather than previous classification-audit fields.

**Expected result**

The dataset is simplified while retaining `PB_or_I`.

df.drop(
    columns=[
        "Classification_Method",
        "Classification_Basis",
        "Classification_Confidence",
        "Classification_Source"
    ],
    errors="ignore",
    inplace=True
)

### Step 22: Final Validation

**What this step does**

The notebook performs two final checks:

1. prints the final PB/I totals, including any remaining missing values;
2. counts rows where both `FMS ID` and `PID` are missing.

**Why this step is needed**

This provides a final data-quality check before the classified file is written to disk.

**Expected result**

You receive a final classification distribution and an identifier-completeness check.

print("\nPB / I totals:")

print(
    df["PB_or_I"]
    .value_counts(
        dropna=False
    )
)

print(
    "\nRows where both FMS ID and PID are missing:",
    (
        df["FMS ID"].isna()
        &
        df["PID"].isna()
    ).sum()
)

### Step 23: Save the Final Classified Dataset

**What this step does**

The completed dataframe is saved to `NYC_Capital_Projects_PB_I_Classified.csv`.

The notebook then prints:

- a completion message,
- final dataset shape,
- output-file location,
- review-file location when unresolved projects remain.

**Why this step is needed**

This creates the final deliverable that can be used for analysis, Tableau, reporting, or presentation.

**Expected result**

The final classified CSV is saved in the configured output folder.

df.to_csv(
    output_file,
    index=False
)


print("CLASSIFICATION COMPLETED")


print(
    "Final dataset shape:",
    df.shape
)

print(
    "Final classified file:",
    output_file
)

if remaining > 0:
    print(
        "Review file:",
        review_file
    )
