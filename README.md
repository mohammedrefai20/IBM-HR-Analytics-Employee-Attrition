# Airbnb Open Data - Power BI Project Documentation

## 1) Dataset Overview and Project Role

This project uses a single source file: `Airbnb_Open_Data.csv`.

- **Rows:** 102,599
- **Columns:** 26
- **Grain (row level):** one Airbnb listing record
- **Primary analytical goal:** analyze listing distribution, pricing, host behavior, review performance, and availability across NYC areas.

### Why this CSV matters in the project

This CSV is the operational core of the report. It contains:
- Listing identity and host identity
- Geographic segmentation (borough + neighborhood + coordinates)
- Pricing and service fee values
- Demand proxies (reviews, review rate, reviews/month)
- Supply constraints (minimum nights, availability)
- Booking policy context (instant bookable + cancellation policy)

Because this is a raw export, it contains quality issues (missing values, inconsistent text, duplicated rows, outliers, invalid ranges). A strong Power Query pipeline and clean star schema are required before building visuals.

---

## 2) Column-by-Column Meaning (Data Dictionary)

| Column | Meaning | Recommended Business Type |
|---|---|---|
| `id` | Listing ID (listing-level unique identifier in source, but duplicates exist in raw data) | Text / Whole Number key |
| `NAME` | Listing title shown to users | Text |
| `host id` | Host identifier | Text / Whole Number key |
| `host_identity_verified` | Whether host profile is verified (`verified`/`unconfirmed`) | Categorical |
| `host name` | Host display name | Text |
| `neighbourhood group` | Borough/group (e.g., Manhattan, Brooklyn) | Categorical |
| `neighbourhood` | Neighborhood inside borough | Categorical |
| `lat` | Latitude coordinate | Decimal |
| `long` | Longitude coordinate | Decimal |
| `country` | Country name (mostly United States) | Categorical |
| `country code` | Country code (mostly US) | Categorical |
| `instant_bookable` | Listing can be instantly booked (`TRUE`/`FALSE`) | Boolean |
| `cancellation_policy` | Cancellation policy category (`flexible`,`moderate`,`strict`) | Categorical |
| `room type` | Inventory type (`Entire home/apt`,`Private room`,`Shared room`,`Hotel room`) | Categorical |
| `Construction year` | Property construction year | Whole Number |
| `price` | Listing price (currency-formatted text in raw data) | Decimal (currency) |
| `service fee` | Service fee (currency-formatted text in raw data) | Decimal (currency) |
| `minimum nights` | Minimum required stay nights | Whole Number |
| `number of reviews` | Total reviews count | Whole Number |
| `last review` | Date of most recent review | Date |
| `reviews per month` | Average monthly review frequency | Decimal |
| `review rate number` | Rating score bucket (1 to 5) | Whole Number |
| `calculated host listings count` | Number of active listings associated with the host | Whole Number |
| `availability 365` | Available nights in a year (expected 0-365) | Whole Number |
| `house_rules` | Free-text listing house rules | Text |
| `license` | License field (almost entirely blank in this file) | Text |

---

## 3) Power Query Data Cleaning and Transformation (Required)

The following steps should be implemented in Power Query in this order.

### 3.1 Import and basic standardization

1. Load `Airbnb_Open_Data.csv` using `UTF-8`.
2. Trim and clean text for all text columns (`Text.Trim`, `Text.Clean`).
3. Standardize column names to modeling-friendly format:
   - Example: `host id` -> `HostID`, `Construction year` -> `ConstructionYear`, `availability 365` -> `Availability365`.

### 3.2 Remove exact duplicates

1. Remove full-row duplicates (`Home > Remove Rows > Remove Duplicates` on all columns).
2. Then remove duplicate listing IDs with a deterministic rule:
   - Sort by `LastReview` descending, then keep first row per `ListingID` (or choose most complete row by non-null count).

> Profiling result from this file: **541 exact duplicate rows** were detected.

### 3.3 Data type fixes

Set explicit types:
- `ListingID`, `HostID`: Text (safer for IDs)
- `Latitude`, `Longitude`, `ReviewsPerMonth`, `Price`, `ServiceFee`: Decimal Number
- `ConstructionYear`, `MinimumNights`, `NumberOfReviews`, `ReviewRateNumber`, `CalculatedHostListingsCount`, `Availability365`: Whole Number
- `LastReview`: Date
- `InstantBookable`: True/False

### 3.4 Currency text cleanup (`price`, `service fee`)

Raw values include `$`, commas, and trailing spaces (example: `"$1,060 "`).

Transform:
1. Replace `$` with empty text.
2. Replace `,` with empty text.
3. Trim spaces.
4. Convert to Decimal Number.

### 3.5 Boolean normalization

`instant_bookable` values are `TRUE`/`FALSE` (text).
- Convert to Boolean.
- Replace null with `false` or `Unknown` (if keeping a category dimension).

### 3.6 Categorical value standardization

Fix typos/inconsistencies:
- `brookln` -> `Brooklyn`
- `manhatan` -> `Manhattan`

Normalize casing for all categories:
- Proper case for borough/neighborhood/room type/policy.

### 3.7 Missing value treatment

Recommended handling by column type:

- **Critical keys (`ListingID`, `HostID`):** remove rows if missing (none expected after import).
- **Location (`NeighbourhoodGroup`, `Neighbourhood`, `Lat`, `Long`):**
  - If coordinates missing, drop rows for map analysis or flag with `GeoValid = 0`.
  - If borough missing but neighborhood exists, infer from lookup table if possible.
- **`LastReview` and `ReviewsPerMonth`:**
  - Keep nulls (they often indicate no reviews yet).
  - Create flag `HasReview = LastReview <> null`.
- **`HouseRules`:**
  - Replace null with `"Not Provided"`.
- **`License`:**
  - Because it is almost fully missing, do not use in KPIs.
  - Keep as informational field only.

### 3.8 Range and outlier validation rules

Apply rule-based cleaning with audit columns:

- `MinimumNights < 1` -> set to null or floor to 1 (business rule needed)
- `Availability365 < 0` or `> 365` -> set to null (invalid range)
- `ReviewRateNumber` outside 1-5 -> set null
- `ConstructionYear` outside plausible range (e.g., 1800-current year) -> set null
- `LastReview` future date -> set null

> Profiling found in this file:
- `minimum nights` has negative values (13 rows)
- `availability 365` has negatives (432 rows) and values above 365 (2,782 rows)
- a few future `last review` dates exist

### 3.9 Add transformation-derived columns in Power Query

Create these columns before model load:
- `PricePerNightNet = Price - ServiceFee`
- `ServiceFeePct = ServiceFee / Price`
- `ListingAge = Date.Year(DateTime.LocalNow()) - ConstructionYear`
- `ReviewRecencyDays = Duration.Days(Date.From(DateTime.LocalNow()) - LastReview)` (null-safe)
- `MinimumNightsBand` (1-3, 4-7, 8-30, 31+)
- `AvailabilityBand` (0, 1-90, 91-180, 181-365)

### 3.10 Keep/Remove columns for analytics

Keep all analytical columns; optionally move long text columns (`HouseRules`) to a detail table if model size/performance becomes an issue.

---

## 4) Data Modeling Design (Star Schema)

Even with one CSV, model it as a star schema for performance and maintainability.

## 4.1 Recommended tables

### Fact table
- **`FactListings`** (one row per listing)
  - Keys: `ListingID`, `HostID`, `LocationKey`, `PolicyKey`, `RoomTypeKey`, `DateKey_LastReview`
  - Numeric fields: `Price`, `ServiceFee`, `MinimumNights`, `NumberOfReviews`, `ReviewsPerMonth`, `ReviewRateNumber`, `Availability365`, etc.

### Dimension tables
- **`DimHost`**
  - `HostID` (PK), `HostName`, `HostIdentityVerified`, `CalculatedHostListingsCount`
- **`DimLocation`**
  - `LocationKey` (PK), `NeighbourhoodGroup`, `Neighbourhood`, `Country`, `CountryCode`, `Latitude`, `Longitude`
- **`DimRoomType`**
  - `RoomTypeKey` (PK), `RoomType`
- **`DimPolicy`**
  - `PolicyKey` (PK), `CancellationPolicy`, `InstantBookable`
- **`DimDate`**
  - Standard date table for `LastReview` (Year, Quarter, Month, MonthName, etc.)

Optional:
- **`DimReviewBand`** (rating bands, review velocity bands)
- **`DimNightsBand`** (minimum-night buckets)

## 4.2 Relationships

Use single-direction, many-to-one relationships from `FactListings` to dimensions:

- `FactListings[HostID]` -> `DimHost[HostID]`
- `FactListings[LocationKey]` -> `DimLocation[LocationKey]`
- `FactListings[RoomTypeKey]` -> `DimRoomType[RoomTypeKey]`
- `FactListings[PolicyKey]` -> `DimPolicy[PolicyKey]`
- `FactListings[LastReviewDate]` -> `DimDate[Date]`

### Key design notes
- Keep surrogate keys in dimensions where possible (`LocationKey`, `PolicyKey`, etc.).
- Preserve original natural keys (`ListingID`, `HostID`) for traceability.
- Avoid bi-directional filtering unless a specific business case requires it.

---

## 5) DAX Measures and Calculated Columns

Below is a complete baseline measure set for a production-quality report.

### 5.1 Core volume and coverage

- `Total Listings = DISTINCTCOUNT(FactListings[ListingID])`
- `Total Hosts = DISTINCTCOUNT(FactListings[HostID])`
- `Total Reviews = SUM(FactListings[NumberOfReviews])`
- `Avg Reviews per Listing = DIVIDE([Total Reviews], [Total Listings])`

Why needed: establishes market size and listing engagement.

### 5.2 Pricing and fee metrics

- `Avg Price = AVERAGE(FactListings[Price])`
- `Median Price = MEDIAN(FactListings[Price])`
- `Min Price = MIN(FactListings[Price])`
- `Max Price = MAX(FactListings[Price])`
- `Total Service Fee = SUM(FactListings[ServiceFee])`
- `Avg Service Fee = AVERAGE(FactListings[ServiceFee])`
- `Service Fee % = DIVIDE([Total Service Fee], SUM(FactListings[Price]))`
- `Avg Net Price = AVERAGE(FactListings[Price] - FactListings[ServiceFee])`

Why needed: compares gross pricing vs effective earnings and fee burden.

### 5.3 Availability and stay constraints

- `Avg Availability Days = AVERAGE(FactListings[Availability365])`
- `Occupancy Proxy % = 1 - DIVIDE([Avg Availability Days], 365)`
- `Avg Minimum Nights = AVERAGE(FactListings[MinimumNights])`
- `High Min Night Listings = CALCULATE([Total Listings], FactListings[MinimumNights] >= 30)`
- `% High Min Night Listings = DIVIDE([High Min Night Listings], [Total Listings])`

Why needed: indicates supply tightness and booking flexibility.

### 5.4 Quality and trust

- `Avg Rating = AVERAGE(FactListings[ReviewRateNumber])`
- `Verified Hosts = CALCULATE([Total Hosts], DimHost[HostIdentityVerified] = "verified")`
- `% Verified Hosts = DIVIDE([Verified Hosts], [Total Hosts])`
- `Instant Bookable Listings = CALCULATE([Total Listings], DimPolicy[InstantBookable] = TRUE())`
- `% Instant Bookable = DIVIDE([Instant Bookable Listings], [Total Listings])`

Why needed: tracks trust signals and booking convenience.

### 5.5 Time intelligence (with `DimDate`)

- `Reviews YTD = TOTALYTD([Total Reviews], DimDate[Date])`
- `Reviews Last 12M = CALCULATE([Total Reviews], DATESINPERIOD(DimDate[Date], MAX(DimDate[Date]), -12, MONTH))`
- `Listings With Recent Review (90D) = CALCULATE([Total Listings], FactListings[ReviewRecencyDays] <= 90)`

Why needed: shows trend momentum and recency of listing activity.

### 5.6 Recommended calculated columns

Use calculated columns only for segmentation logic reused across visuals:

- `Price Band` (`<100`, `100-199`, `200-499`, `500+`)
- `Rating Band` (`Low`, `Mid`, `High`)
- `Availability Band` (`Low`, `Medium`, `High`)
- `Minimum Nights Band`

Prefer Power Query for heavy transformations; use DAX columns for report-specific slicing.

---

## 6) Power BI Report Design (Most Important Section)

Build the report as a multi-page analytical product with clear decision flow.

## 6.1 Report pages and purpose

### Page 1 - Executive Overview (KPI dashboard)

**Purpose:** one-screen business health.

Recommended visuals:
- KPI cards: `Total Listings`, `Total Hosts`, `Avg Price`, `Median Price`, `% Instant Bookable`, `Avg Rating`
- Trend line: `Total Reviews` by month
- Donut/stacked bar: listing share by `Room Type`
- Bar chart: listings by `Neighbourhood Group`
- Map: listing density by coordinates

Expected insight:
- Where supply is concentrated
- Whether market looks premium or budget by area
- How active and trusted listings are overall

### Page 2 - Pricing Intelligence

**Purpose:** explain price behavior.

Recommended visuals:
- Box plot (or custom visual): `Price` by `Neighbourhood Group`
- Heatmap/matrix: `Neighbourhood` x `Room Type` with `Avg Price`
- Scatter: `Price` vs `NumberOfReviews` (size = `Availability365`)
- Decomposition tree: drivers of `Avg Price`

Expected insight:
- High-value neighborhoods
- Which room type drives premium pricing
- Whether high price correlates with demand/reviews

### Page 3 - Host & Policy Analysis

**Purpose:** understand host profile and booking policies.

Recommended visuals:
- Bar chart: verified vs unconfirmed host counts
- Stacked bar: cancellation policy by borough
- KPI: `% Verified Hosts`, `% Instant Bookable`
- Table: top hosts by listing count and average rating

Expected insight:
- Trust and operational profile by location
- Policy strictness and booking friction patterns

### Page 4 - Availability & Demand Signals

**Purpose:** infer demand pressure and listing utilization.

Recommended visuals:
- Histogram: `Availability365` distribution
- Bar: `Minimum Nights Band` share
- Line: `Reviews Last 12M`
- Scatter: `ReviewsPerMonth` vs `Availability365`

Expected insight:
- Potentially high-demand zones (high reviews, lower availability)
- Whether strict minimum nights reduce market accessibility

### Page 5 - Listing Detail Explorer

**Purpose:** drill-through for detailed analysis and quality checks.

Recommended visuals:
- Detail table with conditional formatting:
  - Listing, host, neighborhood, room type, price, rating, availability, minimum nights
- Drill-through from any page by `ListingID` or `HostID`
- Optional text panel for `HouseRules`

Expected insight:
- Row-level explainability and exception investigation.

## 6.2 Slicers and filters (global)

Use top horizontal slicers on every page:
- `Neighbourhood Group`
- `Neighbourhood`
- `Room Type`
- `Cancellation Policy`
- `Host Identity Verified`
- `Instant Bookable`
- `Price Band`
- Date slicer on `Last Review`

Add a reset button/bookmark and synced slicers across pages.

## 6.3 Layout and UX standards

- Use a 12-column grid layout for consistent alignment.
- Keep KPI cards in top row, trend/segmentation visuals in center, detail context at bottom.
- Use consistent color semantics:
  - Positive/trust: green
  - Caution/outlier: amber/red
- Include dynamic titles reflecting filter context.
- Add tooltip pages for borough and neighborhood deep context.

## 6.4 KPI shortlist for stakeholders

Minimum stakeholder-ready KPI set:
- Total Listings
- Active Review Coverage (% with at least one review)
- Avg Price / Median Price
- Avg Rating
- % Verified Hosts
- % Instant Bookable
- Occupancy Proxy %
- Top 5 Neighborhoods by Listings and by Avg Price

---

## 7) Data Quality Notes Observed in This File

From full profiling of `Airbnb_Open_Data.csv`:

- 541 exact duplicate rows exist.
- `license` is effectively unusable (almost all blank).
- Missing `house_rules` is very high (~50%).
- `last review` and `reviews per month` are missing for ~15.5% rows (likely no-review listings).
- Invalid ranges exist:
  - negative `minimum nights`
  - negative and >365 values in `availability 365`
- Category typos exist (`brookln`, `manhatan`).

These issues should be fixed during Power Query preparation before loading to model.

---

## 8) Suggested Project Folder Outputs

Recommended structure after implementation:

- `README.md` (this document)
- `data/` (raw CSV, optional cleaned extracts)
- `powerquery/` (M scripts or transformation notes)
- `model/` (schema diagram image)
- `dax/` (measure definitions)
- `report/` (screenshots or exported PDF of final dashboard)

---

## 9) Final Implementation Checklist

- [ ] Load and profile raw CSV
- [ ] Apply Power Query cleaning pipeline
- [ ] Validate types, ranges, and null handling
- [ ] Build star schema (fact + dimensions)
- [ ] Create DAX measures and segmentation columns
- [ ] Build 4-5 report pages with synchronized slicers
- [ ] Validate KPI outputs against raw aggregates
- [ ] Publish and share with usage notes

