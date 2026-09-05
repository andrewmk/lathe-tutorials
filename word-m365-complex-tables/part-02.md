# Adding Formulas, Styles, and Nested Tables

> [!RECALL]
> Quick recall before we continue: what does `=SUM(ABOVE)` do in a Word table formula, and why does it exclude values in the header row? Try to answer without looking at Part 1.

You've got your timeline table built, headers repeating across pages, cells merged into section banners. It looks good on paper — or rather, on screen. But a project status report isn't just structure; it's data. Your stakeholders need to see numbers that add up without them opening Excel. And the budget table — the one you sketched in your head as a simple grid — needs to distinguish categories with merged cells, flag over-budget items, and sum everything at the bottom.

Part 1 gave you the skeleton. Part 2 gives it a nervous system. You'll add formulas that calculate totals, apply table styles that auto-band rows, nest a second table inside a cell for sub-team assignments, and centre text vertically so your resource allocation grid looks polished instead of lopsided.

By the end of this part, your Q3 Project Status Report will have a working budget table with running totals and conditional flags, and a resource allocation table with nested sub-team grids.

## What you'll build

This part adds the second and third tables from your Q3 Project Status Report:

1. A **budget table** (5 columns × 10 rows) with merged category cells, a formula that totals each row, and a grand total at the bottom.
2. A **resource allocation table** (6 columns × 8 rows) with nested tables inside cells to show sub-team assignments, vertical alignment centred, and text wrapping tuned to prevent awkward breaks.

You'll also apply a table style with banded rows to the timeline table from Part 1 so all three tables share a consistent look.

## Prerequisites

- Completion of Part 1, or at minimum: you know how to insert a table, merge cells, set repeating headers, and split a table.
- The same Word for Microsoft 365 desktop app.

## Budget table: structure and merged categories

Before you can calculate anything, you need the budget table. It has five columns:

| Column | Header | Purpose |
|--------|--------|---------|
| A | Category | Merged cells for grouping |
| B | Item | Line-item description |
| C | Estimated ($) | Budgeted amount |
| D | Actual ($) | Spent amount |
| E | Variance ($) | Formula: Estimated − Actual |

### Create the base table

1. **Insert** → **Table** → **Insert Table…** → **5** columns, **10** rows. Click **OK**.
2. Type the headers into row 1. Make them bold.

### Merge category cells

The first three rows cover "Personnel" (rows 2–4), the next three cover "Equipment" (rows 5–7), and the last three cover "Travel" (rows 8–10). You want the Category column to show the group name once, spanning all items in that group:

1. Select cells A2 through A4.
2. **Table Layout** → **Merge Cells**.
3. Type "Personnel".

Repeat for A5–A7 ("Equipment") and A8–A10 ("Travel").

> [!PREDICT]
> After merging A2:A4 into a single cell, what happens to the data you typed in A3 and A4? Hint: the merged cell keeps the content from the *first* cell you selected (A2). The others are discarded.

### Enter budget data

Type realistic numbers into your table. Here's a sample you can use:

| Category | Item | Estimated ($) | Actual ($) | Variance ($) |
|----------|------|---------------|------------|--------------|
| Personnel | Developer | 45000 | 47200 | |
| Personnel | Designer | 38000 | 36500 | |
| Personnel | PM | 42000 | 41800 | |
| Equipment | Laptops | 12000 | 11500 | |
| Equipment | Servers | 8000 | 9200 | |
| Equipment | Software | 3000 | 2800 | |
| Travel | Conference | 5000 | 6100 | |
| Travel | Client visits | 4000 | 3500 | |
| Travel | Team retreat | 7000 | — | |

For the last cell (Travel / Team retreat / Actual), leave it blank or type a dash. You'll handle both cases in the formula section.

## Adding formulas to the budget table

Word's formula engine is not Excel. It won't handle text, currency symbols, or empty cells gracefully — but for pure numbers, it works well enough for a report like this. The key is knowing the three ways to reference cells and when to use each.

### The Formula command

The Formula dialog lives on the **Table Layout** tab, in the **Data** group. Click **Formula** to open it. You'll see:

- A **Formula** text box where you type or paste a formula.
- A **Number Format** dropdown for formatting the result.
- A **Paste Function** list for common functions.
- A **Paste Bookmark** list for named cell references.

### Positional arguments: =SUM(ABOVE)

The simplest formula uses positional arguments — directions relative to the formula cell. For the Variance column (column E), you want Estimated minus Actual. But there's no positional argument for "subtract the cell to your left from the cell two columns left." Word doesn't support that directly. Instead, you'll use cell references.

> [!HEADS-UP]
> Positional arguments (LEFT, RIGHT, ABOVE, BELOW) only work with functions that aggregate multiple cells: SUM, AVERAGE, COUNT, MAX, MIN, PRODUCT. They do *not* work with arithmetic operators. `=LEFT - RIGHT` is not valid syntax. If you need to subtract one cell from another, you must use cell references (RnCn or A1 notation).

### Cell references: A1 notation

Word supports two reference styles. A1 notation is the one you probably know from Excel:

| Reference | Meaning |
|-----------|---------|
| `A2` | Column A, row 2 |
| `C2` | Column C, row 2 |
| `D2` | Column D, row 2 |
| `C2:D2` | The range from C2 to D2 |

For the Variance formula in cell E2, you want `=C2-D2`:

1. Click cell E2 (the first Variance cell, row 2).
2. **Table Layout** → **Formula**.
3. In the Formula box, type: `=C2-D2`.
4. In the Number Format dropdown, select `$0` (no decimal places).
5. Click **OK**.

The cell now shows the result (in this case, `$-2,200` — negative because Actual exceeded Estimated).

### Copying formulas down

Word doesn't have a "fill down" button. To copy the formula to the remaining rows:

1. Select cell E2 (the one with the formula).
2. **Copy** (`Ctrl+C`).
3. Select cells E3 through E9.
4. **Paste** (`Ctrl+V`).

Each pasted cell automatically adjusts its row reference: E3 becomes `=C3-D3`, E4 becomes `=C4-D4`, and so on.

> [!ASIDE]
> If you're ever unsure what formula a cell contains, click on it and look in the formula bar at the top of the Word window. It shows the raw formula, not the result.

### The grand total row

Below your 10 rows, add one more row (just click in the last cell and press `Tab`). This row holds the totals:

| Category | Item | Estimated ($) | Actual ($) | Variance ($) |
|----------|------|---------------|------------|--------------|
| **Total** | | **=SUM(C2:C10)** | **=SUM(D2:D10)** | **=SUM(E2:E10)** |

For the Estimated total (cell C11), use `=SUM(C2:C10)` — this sums the entire column range. For the Variance total, use `=SUM(E2:E10)`.

> [!HEADS-UP]
> Word formulas **do not update automatically** when you change cell values. They update when:
> - You open the document (all formulas recalculate once)
> - You select the table and press **F9** (updates all formulas in the table)
> - You select a single formula cell and press **F9** (updates that formula)
>
> If you change a budget number and the variance doesn't reflect it, press **F9** on the table. This is the single most common "why didn't it update?" complaint — tell your stakeholders about it upfront.

### Handling empty cells with IF

The last row (Team retreat / Actual) is blank. If you put a formula there, Word will return `0` instead of leaving it empty — because an empty cell evaluates to zero in Word's formula engine. To handle this, use the `DEFINED()` function:

1. In cell D9 (Team retreat / Actual), type `—` (an en-dash, not a number).
2. In cell E9 (Variance), use the formula: `=IF(DEFINED(D9),C9-D9,"—")`.

This reads: "If D9 has a defined value, subtract it from C9; otherwise, show a dash." The `DEFINED()` function returns `1` if the argument evaluates without error, `0` otherwise.

> [!UNVERIFIED]
> I couldn't confirm whether `DEFINED()` works reliably with en-dash characters in Word for Microsoft 365. The function is documented as returning `1` if the argument "has been defined and evaluates without error." An en-dash is text, not a number, so `DEFINED(D9)` might return `1` and then `C9-D9` would fail with a type mismatch. If this doesn't work, the safer approach is to leave the cell truly empty and use `=IF(D9>0,C9-D9,"—")` — the `>0` test implicitly checks for a numeric value. Test this on your version before relying on it in a production document.

## Applying table styles with banded rows

Your budget table works, but it's a wall of white. Banded rows — alternating light shading on even rows — make it readable. Word calls this "table banding," and it's controlled through table styles, not manual shading.

### Understanding table styles

Table styles are the Table Design equivalent of cell styles for paragraphs. A table style is a named collection of formatting rules that applies to different parts of a table:

| Part | What it controls |
|------|-----------------|
| **Header Row** | Font, shading, borders for the first row |
| **Banded Row 1** | Shading for odd rows |
| **Banded Row 2** | Shading for even rows |
| **Total Row** | Formatting for the last row |
| **First Column** | Formatting for column A |
| **Last Column** | Formatting for the last column |

### Applying a built-in style

1. Click anywhere in your budget table.
2. **Table Design** tab → in the **Table Styles** gallery, click the dropdown to expand all styles.
3. Select a style with banded rows — for example, **Table Grid — Accent 2** or **Grid Table 4 — Accent 1**.

The style applies immediately. Odd rows get one shading, even rows get another, and the header row gets a distinct format.

### Customising banding colors

The built-in styles use your document's theme colors. If you want to customise the banding:

1. Click inside the table.
2. **Table Design** → **Table Styles** gallery → right-click the style you want to modify → **Modify Table Style…**.
3. In the dialog, click **Format** → **Alternating Row Colors**.
4. Choose your banding colors — you can set the shade for odd and even rows independently.
5. Click **OK**.

> [!ASIDE]
> If you ever want to remove banding entirely, go to **Table Design** → **Banded Rows** and click it to toggle it off. This doesn't remove the style; it just disables the alternating shading.

### Applying a style to the timeline table

Go back to your timeline table from Part 1. Apply the same table style so all three tables share a consistent look. The style will automatically adapt to the timeline's merged cells and section headers — the shading respects the cell structure, not the grid.

## Nested tables inside cells

Now for the resource allocation table — the one that makes Word tables feel like they're doing something genuinely clever. You'll nest a second table *inside* a cell of the first table. This is how you show hierarchical data without creating a mess of merged cells.

### Create the resource allocation table

1. **Insert** → **Table** → **Insert Table…** → **6** columns, **8** rows. Click **OK**.
2. Type these headers in row 1:

| Column | Header |
|--------|--------|
| A | Resource |
| B | Role |
| C | Team |
| D | Allocation (%) |
| E | Availability |
| F | Notes |

### Enter the main data

Fill in six rows of resource data (rows 2–7), leaving row 8 for a note or summary. Here's sample data:

| Resource | Role | Team | Allocation (%) | Availability | Notes |
|----------|------|------|----------------|--------------|-------|
| Alex Chen | Senior Developer | Backend | 80 | Available | |
| Samira Patel | UI Designer | Frontend | 100 | Full-time | |
| Jordan Lee | QA Engineer | Testing | 60 | Part-time | Starts Oct |
| Morgan Kim | DevOps | Infrastructure | 90 | Available | |
| Taylor Brooks | PM | Planning | 100 | Full-time | |
| Riley Nguyen | Data Analyst | Analytics | 70 | Available | |

### Insert a nested table

Cell C4 (Jordan Lee's Team cell) needs to show sub-team assignments. Instead of typing "QA, UAT, Automation" in one cell, you'll insert a table:

1. Click inside cell C4.
2. **Insert** → **Table** → **Insert Table…** → **3** columns, **2** rows. Click **OK**.
3. Type the sub-team headers in row 1: **Sub-team**, **Members**, **Focus**.
4. Type the sub-team data in row 2: **QA**, "Alex, Samira", "Regression".

You now have a table inside a table. The outer table's cell C4 contains the inner table.

> [!PREDICT]
> When you click inside the nested table, the outer table's ribbons disappear and are replaced by the inner table's ribbons. What happens when you try to select the outer table's cell C4 from the outside? You can't — the nested table "captures" the click. To select the outer cell, click the table selector (the crosshair icon) at the top-left of the outer table, then use the keyboard to navigate.

### Format the nested table

The nested table should be smaller than the outer cell. Set its size:

1. Click inside the nested table.
2. **Table Design** → **Borders** → **No Border** (so the nested table doesn't double-outline the cell).
3. **Table Layout** → **AutoFit** → **AutoFit Contents** (to shrink the nested table to its content).

The result: a compact sub-table inside the cell, with no visible borders, showing structured sub-data.

### Add a second nested table

Cell C2 (Alex Chen's Team) also needs a sub-team breakdown. Repeat the process:

1. Click cell C2.
2. **Insert** → **Table** → **2** columns, **3** rows.
3. Headers: **Area**, **Role**.
4. Data: **API**, "Backend dev"; **Database**, "Schema design"; **CI/CD**, "Pipeline setup".

## Vertical alignment and text wrapping

Your resource allocation table has cells with varying amounts of content. Some cells have one word; others have a full sentence. By default, Word aligns all cell contents to the top. For a polished look, you want to centre everything vertically.

### Set vertical alignment

1. Select all cells in the resource allocation table (click the table selector at the top-left).
2. Right-click → **Table Properties** → **Cell** tab.
3. Under **Vertical alignment**, select **Center**.
4. Click **OK**.

All cell contents are now centred vertically. The cell with one word and the cell with three sentences both sit in the middle of their cell height.

### Prevent awkward line breaks in notes

The Notes column (column F) has longer text. You want it to wrap within the cell, but you also want to prevent Word from breaking a word in the middle if it can avoid it:

1. Select column F.
2. **Table Layout** → **Cell** → **Options**.
3. Check **Wrap text** (to allow wrapping).
4. Check **Allow word wrap** (to allow breaking long words).
5. Click **OK**.

### Set column widths for the resource table

| Column | Width | Mode |
|--------|-------|------|
| A (Resource) | 1.2 in | Inches |
| B (Role) | 1.0 in | Inches |
| C (Team) | 1.5 in | Inches |
| D (Allocation) | 0.8 in | Inches |
| E (Availability) | 0.8 in | Inches |
| F (Notes) | 1.2 in | Inches |

Total: 6.5 inches. Fits comfortably in an 8.5-inch page with 0.5-inch margins.

## Checkpoint

> [!RECALL]
> In the budget table, you used `=C2-D2` in cell E2. If you copy this formula down to E3, what does it become? What about if you used `=C2-D2` and then copied it to a cell in a *different* table — would Word adjust the reference?

**Run this to verify your work so far:**

1. Create a 5×10 budget table with the headers and data listed above.
2. Merge cells A2:A4, A5:A7, and A8:A10 for category grouping.
3. Insert the formula `=C2-D2` in cell E2 and copy it down to E9.
4. Add a grand total row (row 11) with `=SUM(C2:C10)` in C11, `=SUM(D2:D10)` in D11, and `=SUM(E2:E10)` in E11.
5. Press **F9** on the table to recalculate.
6. Create a 6×8 resource allocation table. Insert a nested 3×2 table inside cell C4.
7. Set vertical alignment to Center for all cells in the resource table.
8. Apply a banded table style to both the budget and resource tables.

**Likely errors:**

- *The formula shows #ERROR instead of a number.* One of the cells you're referencing contains text (like a merged category cell). Word can't subtract text from a number. Make sure your formula references only numeric cells.
- *The nested table pushes the outer table wider than expected.* The nested table's AutoFit is set to AutoFit Window instead of AutoFit Contents. Switch to AutoFit Contents.
- *Vertical alignment doesn't seem to work.* You probably have extra paragraph marks (`Enter` keys) inside the cell. Delete them — Word centres the paragraph, not the text, and extra paragraphs create invisible height.

## What's next

In Part 3, you'll tackle cross-table calculations — referencing cells from the budget table inside the timeline table using RnCn notation, building a dashboard cell that pulls the grand total from the budget table into the timeline table's summary column. You'll also explore table sorting, conditional formatting tricks, and how to protect formula cells from accidental edits.

## Exercises

- [ ] Extend the budget table: add a "Notes" column (column F) and use `=IF(E2<0,"Over budget","On track")` in each row to flag over-budget items. Test with at least two over-budget rows.
- [ ] Build a 4×4 table. In cell B2, insert a nested 2×2 table. Set the outer table's cell B2 to have vertical alignment Bottom and left horizontal alignment. Verify the nested table is positioned correctly.
- [ ] Take your timeline table from Part 1. Apply a table style, then customise the banding colours by modifying the style. Change the banded row 1 shading to a light blue and banded row 2 to white.

## Sources

1. [Use a formula in a Word table — Microsoft Support](https://support.microsoft.com/en-us/office/use-a-formula-in-a-word-table-cbd0596e-ea8a-485e-a35d-b2cb2c4f3e27) — Complete reference for Word table formulas: SUM, IF, AVERAGE, COUNT, positional arguments (LEFT, RIGHT, ABOVE, BELOW), RnCn and A1 cell references, and bookmark names.
2. [Cell.Formula method (Word) — Microsoft Learn](https://learn.microsoft.com/en-us/office/vba/api/word.cell.formula) — VBA API reference for inserting formulas programmatically, confirming the Formula and NumFormat parameters.
3. [How to Nest a Table Within a Table in Word — HowToGeek](https://www.howtogeek.com/948/nesting-a-table-inside-a-table-in-word-2007/) — Practical walkthrough of inserting tables inside cells, with tips on border management and sizing.
4. [How to Format Microsoft Word Tables Using Table Styles — Avantix Learning](https://www.avantixlearning.ca/microsoft-word/how-to-format-microsoft-word-tables-using-table-styles/) — Comprehensive guide to table styles, including how to modify styles, customise banding, and apply styles to existing tables.
5. [Set or change table properties — Microsoft Support](https://support.microsoft.com/en-us/office/set-or-change-table-properties-3237de89-b287-4379-8e0c-86d94873b2e0) — Authoritative reference for cell properties including vertical alignment options (Top, Center, Bottom) and cell margin settings.
6. [How to Center Text Vertically in a Word Table — Avantix Learning](https://www.avantixlearning.ca/microsoft-word/how-to-center-text-vertically-in-a-word-table-and-fix-common-issues/) — Troubleshooting guide for vertical alignment, covering the common "extra paragraph mark" pitfall.
