**Task:** Extract the underlying data tables from embedded Tableau Public dashboards on Baruch College's Starr Career Development Center site (post-graduate employment outcomes by major/employer/salary).

**Target pages** [KNOWN, confidence: HIGH — confirmed by direct fetch]:
- https://studentaffairs.baruch.cuny.edu/starr-career-development-center/post-graduate-outcomes/major-related-data/
- https://studentaffairs.baruch.cuny.edu/starr-career-development-center/post-graduate-outcomes/employer-data/
- https://studentaffairs.baruch.cuny.edu/starr-career-development-center/post-graduate-outcomes/salary-data/
- https://studentaffairs.baruch.cuny.edu/starr-career-development-center/post-graduate-outcomes/overall-employment-status-data/
- https://studentaffairs.baruch.cuny.edu/starr-career-development-center/post-graduate-outcomes/internship-data/
- https://studentaffairs.baruch.cuny.edu/starr-career-development-center/post-graduate-outcomes/graduate-school-data/

**What's confirmed:** [KNOWN, confidence: HIGH] The data is not in the static HTML — it's rendered client-side by embedded Tableau visualizations. A plain HTTP fetch only returns placeholder preview images (I saw filenames like `KnowledgeRatebyMajor/Knowledgerate/1_rss.png`, `EmploymentStatusbyMajor/EmpStatusbyMajor/1_rss.png`, `EmploymentTypebyMajor/EmpTypebyMajor/1_rss.png`, `AverageSalarybyMajor/AvgSalarybyMajor/1_rss.png` under a `public.tableau.com/static/images/` path).

[INFERRED, confidence: LOW] Those filename fragments look like Tableau Public's workbook/view naming convention (`{WorkbookName}/{ViewName}`), which would suggest the actual interactive viz lives at URLs like `https://public.tableau.com/views/KnowledgeRatebyMajor/Knowledgerate`. **I have not confirmed this URL resolves — Cowork should verify, not assume.** The real embed name may differ; the first real step is below.

**Step 1 — get ground truth on the embed (don't skip this):**
Have Cowork fetch the *raw* HTML source (not rendered) of each target page and search for a `<div class="tableauPlaceholder">` or `<object class="tableauViz">` tag, or a `<param name='name' value='...'>` field. That gives the exact, confirmed workbook/view path — more reliable than my filename guess.

**Step 2 — recommended tool: `TableauScraper` (Python, PyPI)** [KNOWN, confidence: MED — I recall this package existing and being built for exactly this use case; verify it's still maintained/available before relying on it]
```
pip install tableauscraper --break-system-packages
```
This library handles Tableau Public's undocumented VizQL session-bootstrap handshake (the part that normally requires a live browser + cookies) and returns the underlying crosstab data as pandas DataFrames, given just the public viz URL. Typical usage pattern:
```python
from tableauscraper import TableauScraper as TS

url = "https://public.tableau.com/views/<WorkbookName>/<ViewName>"
ts = TS()
ts.loads(url)
workbook = ts.getWorkbook()
for sheet in workbook.worksheets:
    print(sheet.name)
    print(sheet.data)  # pandas DataFrame of the underlying data
```

**Step 3 — fallback if the library fails or the viz isn't actually on public.tableau.com** (e.g., if it's hosted on Baruch's own Tableau Server instead):
Use Playwright (headless Chromium) to load the page, wait for the Tableau iframe/object to render, then programmatically click the dashboard's native "Download → Crosstab/Data" export button and parse the resulting CSV. This works regardless of hosting, but is slower and more brittle (breaks if Baruch changes the UI).

**Constraint to flag for Cowork:** confirm whether the dashboard is actually on `public.tableau.com` vs. a private/institutional Tableau Server — if it's the latter, `TableauScraper` likely won't authenticate and Playwright + UI-driven export is the only path.

[RULES I BROKE]: The package's current existence/maintenance status is [INFERRED] from training-era knowledge, not verified live — I flagged that explicitly rather than asserting it as current fact.