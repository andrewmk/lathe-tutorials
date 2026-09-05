# Cross-Table References, Sorting, and Finishing the Report

> [!RECALL]
> In Part 2, you used `=C2-D2` to calculate variance in the budget table. What happens to that formula if you select the budget table and press **F9**? Does it recalculate, or does it do nothing?

Your budget table has formulas that add up. Your resource allocation table has nested sub-teams. They're both good on their own, but a project status report is a *single document* — the numbers need to talk to each other. The timeline table should show the total budget spent in its summary column. The budget table should pull resource allocation percentages from the resource table. Right now, they're islands.

Part 3 connects them. You'll use bookmarks and RnCn notation to create cross-table references, build a dashboard cell that pulls the budget grand total into the timeline table, sort the resource allocation table by allocation percentage, and protect your formulas from accidental deletion. By the end, the Q3 Project Status Report will be a single, self-contained document where changing one number cascades through the rest.

By the end of this part, you'll have a dashboard cell in the timeline table that shows the budget grand total from the budget table, a sorted resource table, and a finished document with consistent styling and a summary section.

## What you'll build

This part completes the Q3 Project Status Report:

1. **Cross-table references**: A dashboard cell in the timeline table that displays the budget grand total from the budget table using bookmarks and RnCn notation.
2. **Table sorting**: Sort the resource allocation table by allocation percentage (descending) and by team name (ascending) as a second level.
3. **Formula protection**: Lock the budget table's formula cells so they can't be accidentally deleted.
4. **Document finishing**: Consistent styling across all three tables, a summary section, and final checks.

## Prerequisites

- Completion of Parts 1 and 2, or at minimum: you have a Word document with the three tables (timeline, budget, resource allocation) built as described.
- The budget table has a grand total row with `=SUM(C2:C10)` in the Estimated column.

## Bookmarks: the bridge between tables

Word formulas can only reference cells within the same table. `=SUM(ABOVE)` and `=C2-D2` only work inside the table they're in. To pull a value from the budget table into the timeline table, you need a **bookmark** — a named anchor that Word's formula engine can reference across tables.

### Understanding bookmarks in Word

A bookmark is a named position in a document. Unlike hyperlinks (which jump to a location), bookmarks are invisible and don't appear in the printed document. They exist only as metadata that Word's formula engine can use to locate specific cells.

Think of a bookmark like a street address. The budget table's grand total cell is a house. The timeline table is a different neighbourhood. The bookmark is the address that lets someone in the timeline neighbourhood find the budget house.

### Create a bookmark on the budget grand total

1. Click in cell **C11** of the budget table (the Estimated grand total cell).
2. **Insert** → **Bookmark** (in the Links group on the Insert ribbon).
3. In the Bookmark dialog, type a name: **BudgetTotal**. Click **Add**.

> [!HEADS-UP]
> Bookmark names have strict rules:
> - No spaces (use underscores or camelCase)
> - Must start with a letter
> - Cannot contain punctuation
>
> `BudgetTotal` works. `Budget Total` does not. `1BudgetTotal` does not. If you try to add a bookmark with an invalid name, Word will reject it silently — you'll see the dialog close and nothing will happen. Check that the bookmark was added by reopening the Bookmark dialog; it should appear in the list.

### Reference the bookmark from another table

Now go to the timeline table. You want a dashboard cell in the summary column (column L) that shows the budget total:

1. Click in cell **L20** of the timeline table (the last row, Status column).
2. **Table Layout** → **Formula**.
3. In the Formula box, type: `=BudgetTotal`.

That's it. Just the bookmark name, no equals sign before it (Word adds the `=` automatically when you insert via the Formula dialog). Click **OK**.

The cell now displays the value from the budget table's C11 cell. Change the budget total — press F9 on the budget table — and then press F9 on the timeline table. The dashboard cell updates.

> [!PREDICT]
> If you delete the budget table, what happens to the dashboard cell in the timeline table? Try it.

The dashboard cell will show `#REF!` — Word can't resolve the bookmark because the cell it pointed to no longer exists. Bookmarks don't create a persistent link; they're a snapshot of "this cell at the time the bookmark was created." If the source cell is deleted, the bookmark becomes a dangling reference.

## RnCn notation: precise cell addressing

Bookmarks are great for pulling a single value from another table. But what if you need to reference a specific cell within a table using coordinates? Word supports **RnCn notation**, borrowed from Excel's legacy reference style:

| Reference | Meaning |
|-----------|---------|
| `R1C1` | Row 1, Column 1 |
| `R5C3` | Row 5, Column 3 |
| `R1C` | Entire column of the formula cell (in row 1) |
| `R` | Entire row of the formula cell |
| `R1C1:R5C3` | Range from R1C1 to R5C3 |

### RnCn with bookmarks for cross-table references

When referencing cells in a *different* table, you prefix the RnCn reference with the bookmark name:

| Formula | Meaning |
|---------|---------|
| `BudgetTotal` | The value of the bookmarked cell (simple reference) |
| `BudgetTotal R1C1` | The cell at row 1, column 1 of the bookmarked table |
| `BudgetTotal R1C1:R10C3` | The range from R1C1 to R10C3 of the bookmarked table |

This is how you build a dashboard that pulls structured data from another table. For example, to sum all the Estimated values in the budget table:

1. Create a bookmark on the budget table (select the entire table → Insert → Bookmark → name it `BudgetTable`).
2. In the timeline table, use the formula: `=SUM(BudgetTable R2C3:R10C3)`.

This sums the Estimated column (column C) from rows 2 through 10 of the budget table, *without* needing a grand total row.

> [!ASIDE]
> Bookmarking the entire table (select all cells → Insert → Bookmark) is more robust than bookmarking a single cell. If you add or remove rows in the budget table, the bookmark still covers the whole table. A cell-level bookmark would need to be recreated.

## Sorting tables

Your resource allocation table has six people with different allocation percentages. Right now, they're in the order you typed them. A manager looking at this table wants to see who's most stretched (highest allocation) at the top. Sorting is the quick fix.

### Sort by a single column

1. Select the resource allocation table (click the table selector at the top-left).
2. **Table Layout** → **Sort** (in the Data group).
3. In the Sort dialog:
   - Check **Header row** (so Word doesn't sort your headers into the data).
   - **Sort by**: Column D (Allocation (%)).
   - **Type**: Number.
   - **Order**: Descending.
4. Click **OK**.

The table reorders: Alex (80%) is now first, Samira (100%) is last. Wait — that's wrong. Samira has 100% allocation, so she should be first. Let me correct the sort: **Descending** means highest first, so Samira (100%) should be at the top. If she's not, check that the Allocation column values are actually numbers, not text. Word's number sort will fail silently on text values.

### Sort by multiple columns

You want to sort by Allocation (descending) first, then by Team (ascending) as a tiebreaker:

1. Open the Sort dialog again (**Table Layout** → **Sort**).
2. **Sort by**: Column D (Allocation (%)), **Number**, **Descending**.
3. Click **Then by** → Column C (Team), **Text**, **Ascending**.
4. Click **OK**.

Now if two people have the same allocation percentage, they'll be sorted alphabetically by team name. This is useful when you have multiple people at the same allocation level.

> [!HEADS-UP]
> Word's sort dialog supports up to **three** sort levels. Each level has:
> - A column to sort by (choose from the column headers or type the column number)
> - A Type: **Text**, **Number**, or **Date**
> - An Order: **Ascending** or **Descending**
>
> If your sort doesn't produce the expected result, check the Type setting. Sorting "100" as Text puts it before "80" because "1" comes before "8" alphabetically. Always verify the Type matches your data.

### The Options button

Click **Options** in the Sort dialog to access:

- **Case sensitive**: Treats uppercase and lowercase differently (A before a).
- **Sort language**: Choose the language for alphabetical sorting (important for documents with non-English content).
- **Separate fields at**: For delimited text (like comma-separated values), choose how Word splits the fields.

For the resource allocation table, you don't need any special options. Keep the defaults.

## Protecting formula cells

You've built formulas in the budget table. They work. But what happens when a colleague opens the document, clicks in a formula cell, and presses Delete? The formula vanishes, replaced by a blank cell. The totals are wrong, and nobody notices until the document is printed.

Word doesn't have a "lock cell" feature like Excel. But there are workarounds.

### Method 1: Display field codes as text (not recommended)

You can show the formula as text instead of a result:

1. Click in the formula cell.
2. Press **Alt+F9** (toggles field code display for all fields in the document).
3. The cell now shows `{ =C2-D2 }` instead of the calculated value.

This is not a real protection — anyone can press Alt+F9 again to toggle back. It's more of a visual warning than a lock.

### Method 2: Use document protection (recommended)

Word's document protection can lock specific regions:

1. **Review** → **Protect** → **Restrict Editing**.
2. In the pane that opens on the right, under **Editing restrictions**, select **Allow only this type of editing in the selection**: choose **No changes (read only)**.
3. Click **Yes, Start Protection**.
4. Select the formula cells (E2:E10 and E11 in the budget table).
5. In the Restrict Editing pane, click **Yes, Start Protection** (if not already started), then select the cells and choose **No changes (read only)** from the dropdown.
6. Set a password (optional but recommended).

Now the formula cells are read-only. Anyone trying to edit them will get a prompt asking for the password.

> [!ASIDE]
> Document protection affects the entire document, not just the table. If you protect the budget table's formula cells, those cells are the only ones that remain editable. All other cells in the document become read-only unless you explicitly allow changes to them. Plan your protection strategy before applying it.

### Method 3: The bookmark-and-update workflow

A lighter-weight approach that doesn't require document protection:

1. Bookmark the formula cells (name them something memorable like `BudgetVar1`, `BudgetVar2`, etc.).
2. Tell stakeholders: "If you change any budget numbers, press F9 on the budget table, then press F9 on the timeline table."
3. Add a note above the table: "⚠️ After editing budget values, press F9 to recalculate all totals."

This isn't protection — it's documentation. But it's the approach most people use because Word's formula protection is genuinely limited.

## Building the dashboard

Now let's connect everything. The timeline table needs a summary section at the bottom that pulls data from both the budget table and the resource table.

### Add a summary row to the timeline table

1. Click in the last cell of the timeline table (row 20, column L).
2. Press **Enter** (not Tab — Enter creates a new row).
3. In the new row, merge all 12 cells and type "Summary".
4. Below that, create another row with these cells:

| Column | Content |
|--------|---------|
| A–C | (merged) **Total Budget:** |
| D–F | (merged) |
| G–I | (merged) |
| J–L | **=BudgetTotal** (the bookmark from earlier) |

The last cell pulls the grand total from the budget table. Change a budget number, press F9, and the summary updates.

### Add a resource summary

Similarly, bookmark the resource table's total allocation (you can add a formula in a new row: `=SUM(D2:D7)`) and reference it in the timeline table:

1. In the resource table, add a new row (row 9).
2. In cell D9, insert the formula: `=SUM(D2:D8)`.
3. Bookmark cell D9: **Insert** → **Bookmark** → name it `TotalAllocation`.
4. In the timeline table's summary row, add another cell with the formula: `=TotalAllocation`.

Now the timeline table's summary shows both the total budget and the total allocation percentage.

> [!UNVERIFIED]
> I couldn't confirm whether `=SUM(D2:D8)` works correctly when D8 is a merged cell (since merged cells in Word can behave unpredictably with formulas). If this returns 0 or #ERROR, the workaround is to leave D8 unmerged and use a simple cell reference instead. Test this on your version before relying on it in a production document.

## Finishing the document

All three tables are built and connected. Now make the document look polished.

### Apply consistent table styles

1. Click in the timeline table → **Table Design** → select **Grid Table 4 — Accent 1** (or any style you prefer).
2. Click in the budget table → **Table Design** → select the *same* style.
3. Click in the resource table → **Table Design** → select the *same* style.

All three tables now share the same font, border, and shading rules. The banded rows are consistent across all tables.

### Add a document title

Above the timeline table, add a title:

1. Click above the table (outside any table).
2. Type: **Q3 Project Status Report**.
3. Format it with the **Heading 1** style (Home → Styles → Heading 1).
4. Centre-align it.

### Add a date and author line

Below the title:

1. Type: **Prepared by: [Your Name]** and **Date: [Today's Date]**.
2. Format both as **Normal** style, right-aligned.

### Final column width check

Go through each table and verify the column widths match your design. If you used AutoFit Contents at any point, the widths may have shifted. Set them to Fixed Column Width one last time:

1. Select each table.
2. **Table Layout** → **AutoFit** → **Fixed Column Width**.

## Checkpoint

> [!RECALL]
> You created a bookmark named `BudgetTotal` on cell C11 of the budget table. In the timeline table, you used the formula `=BudgetTotal`. If you change the value in C11 and press F9 on the *timeline* table, does the dashboard cell update? What if you press F9 on the *budget* table instead?

**Run this to verify your work so far:**

1. In the budget table, add a bookmark on cell C11 (the grand total) named `BudgetTotal`.
2. In the timeline table, insert a new row at the bottom. Merge all cells and type "Summary".
3. In the summary row, add a cell with the formula `=BudgetTotal`.
4. Select the resource allocation table and sort it: Allocation (Number, Descending), then Team (Text, Ascending).
5. Apply the same table style to all three tables.
6. Press F9 on the budget table, then F9 on the timeline table. Verify the dashboard cell updated.

**Likely errors:**

- *The dashboard cell shows #REF!* The bookmark was deleted or the source cell was removed. Recreate the bookmark.
- *The sort didn't work as expected.* Check that the Type is set to Number for numeric columns. Text sorting orders "100" before "80".
- *The formula `=SUM(D2:D8)` returns 0.* One of the cells in the range contains text or is part of a merged cell. Check for merged cells in the range.

## What's next

This is the final part. You now have a complete Q3 Project Status Report: a timeline table with repeating headers, a budget table with formulas and cross-table references, and a resource allocation table with nested sub-teams and sorting. The exercises below will push you further — building a dynamic table that grows with data, creating a table of contents that references table cells, and automating table creation with VBA.

## Exercises

- [ ] **Cross-table dashboard**: Bookmark three cells in the budget table (Estimated total, Actual total, Variance total). Create a summary row in the timeline table that pulls all three values using the bookmark references. Verify that changing any budget number cascades through the dashboard.
- [ ] **Multi-level sort**: Build a 5×8 table with columns: Name, Department, Salary, Years, Rating. Sort by Department (Text, Ascending), then Salary (Number, Descending), then Rating (Number, Descending) as the third level. Verify the sort order.
- [ ] **Protected formula table**: Create a 4×6 table with formulas in the last column (`=SUM(ABOVE)` in each row). Use the bookmark-and-update workflow to protect the formulas: bookmark each formula cell, add a note above the table, and test that the formulas update when you press F9.
- [ ] **Nested table with formulas**: In the resource allocation table, create a nested table inside a cell that has its own formula (`=SUM(ABOVE)`). Change a value in the nested table and press F9 on both the nested table and the outer table. Observe how the formulas interact.

## Sources

1. [Use a formula in a Word table — Microsoft Support](https://support.microsoft.com/en-us/office/use-a-formula-in-a-word-table-cbd0596e-ea8a-485e-a35d-b2cb2c4f3e27) — Complete reference for Word table formulas: SUM, IF, AVERAGE, COUNT, positional arguments, RnCn and A1 cell references, bookmark names, and field code update behaviour.
2. [Sort the contents of a table — Microsoft Support](https://support.microsoft.com/en-us/office/sort-the-contents-of-a-table-f8392477-4613-49cd-aba6-7c2e48f1d91f) — Official instructions for sorting tables: header row toggle, sort-by/then-by levels, Type selection (Text/Number/Date), and Options dialog.
3. [Reference to a cell in another table in a Word document from a field — SuperUser](https://superuser.com/questions/571051/reference-to-a-cell-in-an-other-table-in-a-word-document-from-a-field) — Community discussion on cross-table cell references using bookmarks and RnCn notation, including the `{=SUM(TblA A1)}` syntax.
4. [How to Perform Calculations in Microsoft Word — MakeUseOf](https://www.makeuseof.com/how-to-perform-calculations-in-microsoft-word/) — Practical guide to Word calculations: positional arguments, cell references, and cross-table references via bookmarks.
5. [Update fields — Microsoft Support](https://support.microsoft.com/en-us/office/update-fields-7339a049-cb0d-4d5a-8679-97c20c643d4e) — Field code update mechanics: single field (F9), table (select + F9), document (Ctrl+A + F9), and the difference between field results and field codes.
6. [Exploring the Sorting Options in Word: A Comprehensive Guide — Office Watch](https://office-watch.com/2024/sorting-in-word/) — Deep dive into Word's sort dialog: three sort levels, the Using option for tables, case sensitivity, and language settings.
7. [Is there any way to protect formulas in a Word doc table? — Spiceworks Community](https://community.spiceworks.com/t/is-there-any-way-to-protect-formulas-in-a-word-doc-table/885226) — Community discussion on Word's formula protection limitations and VBA-based workarounds.
8. [Cell.Formula method (Word) — Microsoft Learn](https://learn.microsoft.com/en-us/office/vba/api/word.cell.formula) — VBA API reference for inserting formulas programmatically, confirming the Formula and NumFormat parameters.
