# IBM HR Employee Attrition Power BI Guide

This guide is a complete end-to-end Power BI project workflow for the `WA_Fn-UseC_-HR-Employee-Attrition.csv` dataset. It is written for a junior analyst and uses only the Power BI Desktop interface and Power Query Editor menus. No code or M language is required.

## 1. Data & Domain Understanding

### Business context
Employee attrition is the voluntary or involuntary loss of employees from an organization. In HR analytics, the goal is to understand the drivers behind attrition so business leaders can improve retention, reduce recruitment costs, and maintain productivity. This dataset contains employee records from an IBM sample HR analytics project, including demographic, job, compensation, satisfaction, and career history details.

### Target variable
- `Attrition` – This is the target variable. It contains `Yes` or `No` values.
- Use it to measure attrition rate, compare groups, and build attrition-focused reports.

### Column categories
The dataset has 35 columns. Grouping them into logical categories helps structure cleaning, modeling, and reporting.

#### Demographics
- `Age`
- `Gender`
- `MaritalStatus`
- `DistanceFromHome`
- `Over18` (constant)
- `EmployeeCount` (constant)
- `StandardHours` (constant)
- `EmployeeNumber` (unique row identifier)

#### Job information
- `BusinessTravel`
- `Department`
- `JobRole`
- `JobLevel`
- `JobInvolvement`
- `JobSatisfaction`
- `EnvironmentSatisfaction`
- `RelationshipSatisfaction`
- `WorkLifeBalance`
- `PerformanceRating`
- `OverTime`

#### Compensation
- `DailyRate`
- `HourlyRate`
- `MonthlyIncome`
- `MonthlyRate`
- `PercentSalaryHike`
- `StockOptionLevel`

#### Satisfaction & performance
- `EnvironmentSatisfaction`
- `JobSatisfaction`
- `RelationshipSatisfaction`
- `WorkLifeBalance`
- `JobInvolvement`
- `PerformanceRating`

#### Career history
- `NumCompaniesWorked`
- `TotalWorkingYears`
- `TrainingTimesLastYear`
- `YearsAtCompany`
- `YearsInCurrentRole`
- `YearsSinceLastPromotion`
- `YearsWithCurrManager`

### Columns to remove
These columns are constant across every row and are usually safe to remove:
- `EmployeeCount` = 1 for every employee
- `StandardHours` = 80 for every employee
- `Over18` = Y for every employee

### Special notes
- `EmployeeNumber` is a unique identifier for each record. Keep it as the row-level key in the main fact table.
- `Attrition` is the binary outcome used for reporting and calculations.
- The six satisfaction/rating fields are numeric codes. Use readable label columns for clarity in reports.

---

## 2. Data Cleaning in Power BI Desktop

### Step 1: Load the CSV file
1. Open Power BI Desktop.
2. On the ribbon, go to `Home` → `Get Data`.
3. Select `Text/CSV` and click `Connect`.
4. Browse to `Dataset/WA_Fn-UseC_-HR-Employee-Attrition.csv` and click `Open`.
5. In the preview window, click `Transform Data` to open Power Query Editor.

### Step 2: Remove redundant columns
1. In Power Query Editor, hold `Ctrl` and click the following column headers:
   - `EmployeeCount`
   - `StandardHours`
   - `Over18`
2. Right-click any selected header and choose `Remove`.
3. Optionally remove `EmployeeNumber` from the report visuals later if you do not need it.

### Step 3: Verify and correct data types
1. In Power Query Editor, look at the type icon (ABC, 123, calendar, etc.) on each column header.
2. If any column has the wrong type, click the type icon and select the correct type.
3. Recommended types:
   - `Age` → Whole Number
   - `Attrition` → Text
   - `BusinessTravel` → Text
   - `DailyRate`, `HourlyRate`, `MonthlyIncome`, `MonthlyRate` → Whole Number
   - `DistanceFromHome` → Whole Number
   - `Education` → Whole Number
   - `EducationField` → Text
   - `EnvironmentSatisfaction`, `JobInvolvement`, `JobLevel`, `JobSatisfaction`, `RelationshipSatisfaction`, `WorkLifeBalance`, `PerformanceRating`, `StockOptionLevel`, `TrainingTimesLastYear`, `YearsAtCompany`, `YearsInCurrentRole`, `YearsSinceLastPromotion`, `YearsWithCurrManager`, `NumCompaniesWorked`, `TotalWorkingYears` → Whole Number
   - `Department`, `Gender`, `JobRole`, `MaritalStatus`, `OverTime` → Text

### Step 4: Check for nulls and blank rows
1. In Power Query Editor, go to the `View` tab.
2. Enable `Column quality`, `Column profile`, and `Column distribution`.
3. Inspect the top of each column for any `Error` or `Empty` markers.
4. If there are blank or null rows, go to the `Home` tab.
5. Select `Remove Rows` → `Remove Blank Rows`.

### Step 5: (Your actual workflow) Ratings replaced in-place instead of new label columns
Note: you indicated you changed approach for the satisfaction/rating fields. Instead of adding new label columns, you converted those numeric columns to text and replaced their numeric codes with readable text labels directly. The steps below document that in-UI workflow and the implications for modeling.

1. In Power Query Editor, select one of the numeric satisfaction/rating columns (for example `JobSatisfaction`).
2. Click the type icon on the column header and choose `Text` to change the column's data type to `Text`.
3. With the column still selected, go to the `Transform` tab (or right-click the column header) and choose `Replace Values`.
4. In the `Replace Values` dialog set `Value To Find` to the numeric code (for example `1`) and `Replace With` to the label (for example `Low`). Click `OK`.
5. Repeat `Transform` → `Replace Values` for each numeric code you want to replace (e.g., `2` → `Medium`, `3` → `High`, `4` → `Very High`).
6. Repeat steps 1–5 for each of the satisfaction/rating fields you changed (these include `EnvironmentSatisfaction`, `JobSatisfaction`, `RelationshipSatisfaction`, `WorkLifeBalance`, `JobInvolvement`, and `PerformanceRating`).

Important note: By changing the original columns to `Text` and replacing values in-place, you have overwritten the numeric codes in those columns. If you want to keep numeric versions for aggregation or averaging, create a `Reference` query later and cast the field back to `Whole Number` there (instructions in the Data Modeling section). Otherwise use these text-labeled columns as categorical labels and rely on other numeric fields (or recreated numeric copies) for measures.

### Step 6: Your created columns — `IncomeBand` and `AgeBand`
You mentioned you only created `IncomeBand` and `AgeBand` and kept the original columns.

Create `IncomeBand` (UI steps):
1. In Power Query Editor go to `Add Column` → `Conditional Column`.
2. Set `New column name` to `IncomeBand`.
3. Under `Column name`, choose `MonthlyIncome`.
4. Add clauses exactly as you prefer (example):
   - If `is less than or equal to` `3000` then `Low`
   - Else if `is less than or equal to` `6000` then `Medium`
   - Else if `is less than or equal to` `9000` then `High`
   - Else `Very High` (default `Else` output)
5. Click `OK`.

Create `AgeBand` (UI steps):
1. In Power Query Editor select `Age` and go to `Add Column` → `Conditional Column`.
2. Set `New column name` to `AgeBand`.
3. Add clauses for bins (example 5-year bins):
   - If `is less than or equal to` `24` then `20-24`
   - Else if `is less than or equal to` `29` then `25-29`
   - Continue similarly for other ranges.
4. Click `OK`.

Because you kept the original columns, you can use numeric `Age`/satisfaction columns for calculations (if still numeric) or the text replacements / bands for labeling and slicers.

### Step 7: Rename columns for readability
1. In Power Query Editor, double-click any column header text.
2. Type the new name and press `Enter`.
3. Recommended rename examples:
   - `EmployeeNumber` → `EmployeeID` (optional)
   - `NumCompaniesWorked` → `CompanyCount`
   - `YearsInCurrentRole` → `YearsInRole`
   - `YearsWithCurrManager` → `YearsWithManager`
4. Keep names consistent and user-friendly.

### Step 8: Apply changes
1. In Power Query Editor, go to the `Home` tab.
2. Click `Close & Apply`.
3. Wait for Power BI to apply the changes and return to the report view.
### Step 9: Rename columns for readability
1. In Power Query Editor, double-click any column header text.
2. Type the new name and press `Enter`.
3. Recommended rename examples:
   - `EmployeeNumber` → `EmployeeID` (optional)
   - `NumCompaniesWorked` → `CompanyCount`
   - `YearsInCurrentRole` → `YearsInRole`
   - `YearsWithCurrManager` → `YearsWithManager`
4. Keep names consistent and user-friendly.

### Step 10: Apply changes
1. In Power Query Editor, go to the `Home` tab.
2. Click `Close & Apply`.
3. Wait for Power BI to apply the changes and return to the report view.

---

## 3. Data Modeling

### Recommended schema
Use a star schema with one central fact table and supporting dimension tables.

- Fact table: `FactEmployee`
- Dimension tables:
  - `DimDepartment`
  - `DimJobRole`
  - `DimEducation`
  - `DimSatisfaction`

### Create the main fact table
1. In Power Query Editor, rename the main query to `FactEmployee`.
2. Keep all cleaned columns in this query.
3. Ensure the fact table includes `EmployeeID`, `Attrition`, `AttritionFlag`, numeric measures, and dimension keys.

### Create dimension tables by referencing the main query (recommended)
Use `Reference` queries so the main `FactEmployee` stays as the authoritative table and each dimension is derived from it. This keeps the ETL logic centralized and avoids accidental divergence.

Steps to create a dimension via `Reference` (UI-only):
1. In Power Query Editor, right-click the `FactEmployee` query and choose `Reference` (not `Duplicate`). This creates a new query that points to the output of `FactEmployee`.
2. Rename the new query to the intended dimension name (for example `DimDepartment`).
3. With `DimDepartment` selected, right-click on the `Department` column header and choose `Remove Other Columns` to keep only `Department`.
4. On the `Home` tab, choose `Remove Rows` → `Remove Duplicates` to create a unique list of departments.
5. Verify the data type by clicking the type icon on `Department` and set it to `Text`.
6. Rename the column if desired (double-click header).
7. Repeat these steps for other dimensions.

Suggested dimension contents and exact UI steps:

#### DimJobRole
- Right-click `FactEmployee` → `Reference` → rename `DimJobRole`.
- Right-click `JobRole` → `Remove Other Columns`.
- Optionally keep `JobLevel` and `Department`: select `JobRole`, `JobLevel`, `Department` then right-click → `Remove Other Columns`.
- `Home` → `Remove Rows` → `Remove Duplicates` on `JobRole`.
- Check types (type icon) and rename columns via double-click.

#### DimEducation
- Right-click `FactEmployee` → `Reference` → rename `DimEducation`.
- Keep `Education`, `EducationField`, and if you created one, `EducationLabel` (or the replaced text values).
- `Home` → `Remove Rows` → `Remove Duplicates` on `Education`.
- Verify types and rename as needed.

#### DimSatisfaction (optional)
- Right-click `FactEmployee` → `Reference` → rename `DimSatisfaction`.
- Keep the satisfaction label columns you created or the satisfaction fields you replaced in-place:
   `EnvironmentSatisfaction`, `JobSatisfaction`, `RelationshipSatisfaction`, `WorkLifeBalance`, `JobInvolvement`, `PerformanceRating`.
- `Home` → `Remove Rows` → `Remove Duplicates` to get distinct combinations used for slicers/legends.

Tip about your in-place replacements: because you converted satisfaction fields to `Text` and replaced codes in the main table, you can use those text fields directly in your dimension lookups for labels and slicers. If you later need numeric aggregations (for example average job satisfaction), create a `Reference` of `FactEmployee` and in that referenced query change the satisfaction field's type back to `Whole Number` via the type icon, then `Close & Apply` — use that numeric version only for measures.

### Load dimension tables
1. After preparing each referenced query, click `Close & Apply` on the `Home` tab in Power Query Editor.
2. Power BI will load the fact and dimension tables into the data model.

### Set up relationships in Model view (UI step-by-step)
1. Go to the `Model` view in Power BI Desktop (left-hand vertical bar, the model icon).
2. If autosaved relationships are not present, manually create them by dragging fields:
    - Click `FactEmployee[Department]` and drag to `DimDepartment[Department]`.
    - Click `FactEmployee[JobRole]` and drag to `DimJobRole[JobRole]`.
    - Click `FactEmployee[Education]` and drag to `DimEducation[Education]`.
    - For satisfaction labels, drag `FactEmployee[JobSatisfaction]` (or the label field) to `DimSatisfaction[JobSatisfaction]`.
3. After each relationship is created, click the relationship line to open the properties pane.
    - Confirm `Cardinality` is `Many to one (*:1)`.
    - Set `Cross filter direction` to `Single` (from dimension to fact) unless you specifically need both directions.
    - If `EmployeeID` is used as a primary key in `FactEmployee`, do not create a dimension on it; it stays as a fact key.
4. Alternatively use `Manage relationships` (Modeling → Manage relationships) to edit cardinality and filter directions explicitly.

### Relationship table (expanded)
| Fact table column | Dimension table | Dimension key | Cardinality | Cross-filter direction |
|-------------------|-----------------|---------------|-------------|------------------------|
| `FactEmployee[Department]` | `DimDepartment` | `DimDepartment[Department]` | Many-to-one (*:1) | Single |
| `FactEmployee[JobRole]` | `DimJobRole` | `DimJobRole[JobRole]` | Many-to-one (*:1) | Single |
| `FactEmployee[Education]` | `DimEducation` | `DimEducation[Education]` | Many-to-one (*:1) | Single |
| `FactEmployee[JobSatisfaction]` (label) | `DimSatisfaction` | `DimSatisfaction[JobSatisfaction]` | Many-to-one (*:1) | Single |

> Note: If you need to perform numeric aggregations on satisfaction, create a numeric copy in a referenced query solely for measures instead of overwriting the main table values.

---

## 4. DAX Measures

Create the following measures on the `FactEmployee` table using `Modeling` → `New measure`.

### Total Employees
```DAX
Total Employees = COUNTROWS(FactEmployee)
```
- Counts all rows in the employee fact table.

### Attrition Count
```DAX
Attrition Count = CALCULATE(
    COUNTROWS(FactEmployee),
    FactEmployee[Attrition] = "Yes"
)
```
- Counts employees whose `Attrition` value is `Yes`.

### Attrition Rate (%)
```DAX
Attrition Rate (%) =
DIVIDE(
    [Attrition Count],
    [Total Employees],
    0
) * 100
```
- Calculates the percent of employees who attrited.

### Average Monthly Income
```DAX
Average Monthly Income = AVERAGE(FactEmployee[MonthlyIncome])
```
- Calculates the average monthly income across the selected employees.

### Average Age
```DAX
Average Age = AVERAGE(FactEmployee[Age])
```
- Calculates the average age.

### Average Job Satisfaction
```DAX
Average Job Satisfaction = AVERAGE(FactEmployee[JobSatisfaction])
```
- Calculates the average numeric job satisfaction level.

### Overtime Attrition Rate
```DAX
Overtime Attrition Rate =
CALCULATE(
    [Attrition Rate (%)],
    FactEmployee[OverTime] = "Yes"
)
```
- Measures attrition rate for employees who work overtime.

### Non-Overtime Attrition Rate
```DAX
Non-Overtime Attrition Rate =
CALCULATE(
    [Attrition Rate (%)],
    FactEmployee[OverTime] = "No"
)
```
- Measures attrition rate for employees who do not work overtime.

### Dynamic Title
```DAX
Dynamic Title =
"Attrition Overview for " &
IF(
    HASONEVALUE(DimDepartment[Department]),
    VALUES(DimDepartment[Department]),
    "All Departments"
)
```
- Creates a dynamic report title based on the selected department.

### High Risk Flag
```DAX
High Risk Flag =
IF(
    AND(
        SELECTEDVALUE(FactEmployee[OverTime]) = "Yes",
        SELECTEDVALUE(FactEmployee[JobSatisfaction]) <= 2,
        SELECTEDVALUE(FactEmployee[YearsAtCompany]) <= 3
    ),
    1,
    0
)
```
- Flags employees who are working overtime, have low job satisfaction, and are early in their tenure.

---

## 5. Reports & Visualizations

Create a three-page report with consistent formatting and slicer interaction.

### Common report setup
- Use a consistent color theme:
  - Red for attrition-related metrics and visuals.
  - Green for retention or positive outcomes.
  - Blue for neutral or supporting visuals.
- Use `View` → `Sync slicers` to sync slicers across all report pages.
- Use `View` → `Mobile Layout` to build a mobile version after the main report is complete.
- Right-click each visual and choose `Edit alt text` to add accessibility descriptions.
- When finished, publish with `Home` → `Publish`.

### Page 1: Attrition Overview

#### KPI cards
1. Create four `Card` visuals from the Visualizations pane.
2. Drag the following measures into each card:
   - `Total Employees`
   - `Attrition Count`
   - `Attrition Rate (%)`
   - `Average Monthly Income`
3. Format each card in the `Format` pane with a clear title and data label.
4. Use red data color for attrition measures and blue/green for positive totals.

#### Donut chart by gender
1. Add a `Donut chart` visual.
2. Drag `Gender` to `Legend`.
3. Drag `Attrition Count` to `Values`.
4. In the `Format` pane, turn on `Detail labels` and choose `Category, data value`.
5. Sort by `Attrition Count` in descending order.

#### Clustered bar chart by department
1. Add a `Clustered bar chart` visual.
2. Drag `Department` to `Axis`.
3. Drag `Attrition Rate (%)` to `Values`.
4. In the `Format` pane, set `Data colors` to a red palette.
5. Sort the chart by `Attrition Rate (%)` descending.

#### Slicer panel
1. Add four `Slicer` visuals.
2. Drag these fields into them:
   - `Department`
   - `Gender`
   - `OverTime`
   - `Attrition`
3. For each slicer, use the dropdown in the visual header to select `Dropdown` or `List` layout.
4. Use `View` → `Sync slicers` to make these slicers available on every page.

### Page 2: Employee Demographics & Satisfaction

#### Age histogram with 5-year bins
1. In the Fields pane, right-click `Age` and choose `New group`.
2. Choose `Bins` and set `Bin size` to `5`.
3. Add a `Clustered column chart` visual.
4. Drag the new `Age (bins)` field to `Axis`.
5. Drag `EmployeeNumber` to `Values` and ensure it is summarized as `Count`.
6. In `Format`, set axis titles and data colors to blue.

#### Satisfaction by job role bar chart
1. Add a `Stacked bar chart` visual.
2. Drag `JobRole` to `Axis`.
3. Drag `Average Job Satisfaction` measure to `Values`.
4. In `Format`, turn on `Data labels` and set `Color` to green.
5. Sort by `Average Job Satisfaction` descending.

#### MonthlyIncome vs. Attrition scatter plot
1. Add a `Scatter chart` visual.
2. Drag `MonthlyIncome` to `X Axis`.
3. Drag `AttritionFlag` to `Y Axis`.
4. Drag `Attrition` to `Legend`.
5. Drag `EmployeeNumber` to `Details`.
6. In the `Format` pane, set `X-axis` and `Y-axis` titles.
7. Open the `Analytics` pane and add a `Y constant line` at `0.5` to visually separate attrition from retention.
8. In the `Tooltip` field, add `Department`, `JobRole`, `MonthlyIncome`, and `Attrition`.

#### WorkLifeBalance × JobSatisfaction matrix
1. Add a `Matrix` visual.
2. Drag `WorkLifeBalance` to `Rows`.
3. Drag `JobSatisfaction` to `Columns`.
4. Drag `EmployeeNumber` to `Values` and use `Count`.
5. In `Format`, expand `Conditional formatting` on `Values`.
6. Turn on `Background color` and choose a red-to-green diverging palette.
7. Optionally turn on `Data bars` for an additional visual cue.

### Page 3: Risk Factors Deep Dive

#### Overtime vs. attrition stacked bar
1. Add a `Stacked column chart` visual.
2. Drag `OverTime` to `Axis`.
3. Drag `Attrition` to `Legend`.
4. Drag `EmployeeNumber` to `Values` and summarize as `Count`.
5. In `Format`, set `Data colors` with red for `Yes` and green for `No`.

#### Years at company line chart with shaded early-tenure area
1. Add a `Line chart` visual.
2. Drag `YearsAtCompany` to `Axis`.
3. Drag `Attrition Rate (%)` to `Values`.
4. In the `Format` pane, use a blue line color.
5. If you want a shaded early-tenure region, insert a shape: go to `Insert` → `Shapes` → `Rectangle`.
6. Place the rectangle behind the first 1–3 years of the chart, set `Fill` to a light color, and set `Transparency` to 70%.
7. Add a small text box label over the shaded area that reads `Early-tenure risk zone`.

#### High risk employee table with conditional formatting
1. Add a `Table` visual.
2. Drag these fields into the table:
   - `EmployeeNumber`
   - `Age`
   - `Department`
   - `JobRole`
   - `OverTime`
   - `JobSatisfaction`
   - `AttritionFlag`
   - `IncomeBand`
3. In `Format`, expand `Conditional formatting`.
4. Apply `Background color` to `AttritionFlag` using red for `1` and green for `0`.
5. Apply `Font color` or `Data bars` to `JobSatisfaction` so low values stand out.

#### Business travel column chart
1. Add a `Clustered column chart` visual.
2. Drag `BusinessTravel` to `Axis`.
3. Drag `Attrition Count` to `Values`.
4. In `Format`, set `Data colors` to red for attrition-focused analysis.

#### Top-N range slicer
1. Add a `Slicer` visual.
2. Drag `MonthlyIncome` to the slicer.
3. In the slicer header, choose `Between`.
4. Use the range handles to filter the report to the top earners or any income range.
5. Label this slicer `Income range`.

#### Distance from home scatter plot
1. Add another `Scatter chart` visual.
2. Drag `DistanceFromHome` to `X Axis`.
3. Drag `AttritionFlag` to `Y Axis`.
4. Drag `MonthlyIncome` to `Size`.
5. Drag `Attrition` to `Legend`.
6. In the `Tooltip` field, add `Department`, `JobRole`, `DistanceFromHome`, `MonthlyIncome`, and `Attrition`.
7. In `Format`, turn on `Data labels` if needed.

---

## Final steps

- Review all pages for consistent colors and fonts.
- Use `View` → `Sync slicers` to synchronize slicers across Page 1, Page 2, and Page 3.
- Use `View` → `Mobile Layout` to create a mobile version and arrange the most important visuals for phone view.
- For accessibility, right-click each visual and choose `Edit alt text`. Provide a short description such as `Attrition rate by department`.
- Save the Power BI file.
- Publish the report with `Home` → `Publish`.

This README gives you a complete, UI-first Power BI workflow from raw CSV to published report without writing any code.