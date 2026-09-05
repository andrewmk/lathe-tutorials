# Word for Windows M365: Structure, Citations, and Collaboration

> [!RECALL]
> Quick recall before we continue: what is a section break, and why does it let you have different page orientations on different pages of the same document? Name the two things that are independent per section.

You finished Part 2 with a document that has proper margins, headers, footers, page numbers, and section breaks. But the document is still a flat stack of content — no one can navigate it easily, no one has reviewed it, and there's no way to cite sources or merge revisions. This part adds the tools that turn a document from a static artifact into a living, collaborative object.

By the end of this part, you will have:

- Built an automatic table of contents that reads your heading hierarchy
- Learned the Navigation Pane, which gives you a clickable outline of your document
- Added a citation and a works cited list using Word's reference manager
- Turned on Track Changes and added a comment to your document
- Compared two versions of a document to see what changed

The version this tutorial targets is **Word for Microsoft 365, version 2605 (build 20026.20140)**. All interface elements referenced are on the View tab, References tab, or Review tab.

## The Navigation Pane: your document's outline

Before you build a table of contents, you need to see your document's structure. The **Navigation Pane** is a side panel that shows your document as a clickable outline — every heading, every section, laid out in a tree.

On the **View** tab, in the **Show** group, check the box for **Navigation Pane**. The pane slides open on the left side of the window. It has three tabs:

- **Heading** — shows your document's headings in a collapsible tree. Click any heading to jump to that location.
- **Pages** — shows thumbnails of every page. Click a thumbnail to jump to that page.
- **Results** — searches the document for text (covered later).

> [!PREDICT]
> If you click a heading in the Navigation Pane, what do you think happens? Does it just scroll there, or does it select the text?

Click on any heading in the Navigation Pane. Word jumps to that location in the document. This is the fastest way to navigate a long document — no scrolling, no hunting. You can also drag and drop headings in the Navigation Pane to reorder sections (Word moves the content and any sub-headings with them).

> [!ASIDE]
> The Navigation Pane is not a table of contents. It reads your heading styles and builds a tree, but it doesn't show page numbers or tab leaders. A TOC is a printed element; the Navigation Pane is a working tool. Use both.

## Building a table of contents

A table of contents (TOC) in Word is **automatic** — it reads your heading styles and builds a formatted list of headings with page numbers. You don't type it. You don't update it manually. You insert it, and Word manages it.

Place your cursor at the beginning of your document (before the title, or after the title on a new page). On the **References** tab, in the **Table of Contents** group, click **Table of Contents**. A dropdown appears with several built-in TOC styles and an option for **Custom Table of Contents**.

Click the first style (the one with dotted tab leaders). Word inserts a formatted table of contents at your cursor position. It lists every Heading 1, Heading 2, and Heading 3 in your document, with page numbers aligned on the right, connected by dotted tab leaders.

> [!HEADS-UP]
> The TOC is a **field** — a dynamic element that Word updates, not static text. If you edit your document and headings change, the TOC won't update automatically. You must click **Update Table** to refresh it. This is the single most common mistake Word users make: editing the TOC directly instead of updating it.

> [!TIP]
> Don't try to edit the TOC by clicking on it and typing. Word will show a dialog box warning that the change will be lost when you update the table. If you need to customize the TOC, use the **Custom Table of Contents** dialog instead.

### Updating the table of contents

Add some new content to your document — a new section with a Heading 2 subheading. Now click inside the TOC and click **Update Table** (it appears above the TOC when selected, or you can right-click the TOC and choose **Update Field**). You'll see two options:

- **Update page numbers only** — keeps the existing headings but refreshes the page numbers. Use this when you've added or removed content but haven't changed any headings.
- **Update entire table** — rebuilds the entire TOC, including any new headings or removed headings. Use this when you've added or removed heading styles.

> [!PREDICT]
> If you add a new Heading 3 deep in your document and click "Update page numbers only," will the new heading appear in the TOC? Why or why not?

### Customizing the table of contents

Click **Table of Contents > Custom Table of Contents**. A dialog box opens with several options:

- **Show page numbers** — toggle page numbers on or off
- **Right align page numbers** — align page numbers to the right margin (connected by tab leaders)
- **Include level** — choose how many heading levels to show (default is 3: Heading 1, 2, and 3)
- **Tab leader** — choose the character that connects headings to page numbers (dots, dashes, or none)
- **Formats** — choose a preset style (from Modern, Classic, or Formal)

> [!ASIDE]
> The "Formats" dropdown doesn't change your document's styles — it changes the TOC's own formatting. The TOC is formatted using built-in styles called TOC 1, TOC 2, TOC 3, etc. You can modify these styles to change the TOC's appearance without touching your document's heading styles.

## Citing sources: Word's reference manager

Word has a built-in reference manager that lets you add citations, manage sources, and generate a works cited list or bibliography. It supports multiple citation styles: APA, MLA, Chicago, Harvard, and many more.

> [!HEADS-UP]
> Word's citation manager is basic. It's fine for simple papers, but if you're writing a thesis or a book, you'll want a dedicated reference manager like Zotero, Mendeley, or EndNote. Word can integrate with some of them, but the built-in manager is the starting point.

### Adding a source

Place your cursor at the end of the sentence where you want to insert a citation. On the **References** tab, in the **Citations & Bibliography** group:

1. Click the **Style** dropdown and select a citation style (e.g., **APA** or **MLA**).
2. Click **Insert Citation > Add New Source**.
3. A dialog box opens. Fill in the source details:
   - **Author** — name of the author(s)
   - **Title** — title of the work
   - **Year** — publication year
   - **Source type** — book, website, journal, etc.
4. Click **OK**.

Word inserts an in-text citation (e.g., `(Smith, 2024)`) at your cursor position and saves the source in your document's source list.

> [!ASIDE]
> The source list is stored in the document itself. If you share the document with someone, they get your sources too. You can also save sources to a master list by clicking **Manage Sources** and choosing **Master List**.

### Generating a works cited list

Place your cursor at the end of your document (on a new page). On the **References** tab, click **Bibliography**. A dropdown appears with several bibliography styles. Click the first one (Works Cited).

Word generates a formatted works cited list at your cursor position, using the sources you've added. The list is alphabetized by author and formatted according to the citation style you selected.

> [!PREDICT]
> If you add a new source after generating the works cited list, will it appear in the list automatically? What do you need to do to update it?

### Updating the bibliography

Just like the table of contents, the bibliography is a field. If you add, remove, or edit sources, click inside the bibliography and click **Update Bibliography** (or right-click and choose **Update Field**).

## Track Changes: editing with a paper trail

**Track Changes** records every edit you make — every insertion, deletion, formatting change — so you can see what changed and who changed it. This is essential for collaborative documents, legal documents, and any document that goes through multiple rounds of review.

On the **Review** tab, in the **Tracking** group, click **Track Changes** to turn it on. The button will be highlighted (or shaded) to indicate it's active.

Now make some edits to your document:

- Type some new text
- Delete some existing text
- Change a word's formatting (bold, italic, etc.)

Every change is marked with **revision marks**:

- **Insertions** appear in blue (by default) and are underlined
- **Deletions** appear in blue (by default) and are struck through
- **Formatting changes** appear in a separate markup layer

> [!ASIDE]
> The color for each author's changes is assigned automatically. If multiple people are editing, each gets a different color. Word shows the author name in the status bar when Track Changes is on.

### Showing and hiding revisions

By default, Word shows all revisions in the document. You can control how they're displayed:

- **All Markup** — shows all revisions with markup visible
- **No Markup** — shows the document as if all revisions were accepted (but doesn't accept them)
- **Original** — shows the document before any revisions
- **Simple Markup** — shows a vertical line in the margin instead of inline markup (like Word's default "balanced" view)

On the **Review** tab, click **Display for Review** to switch between these views.

> [!TIP]
> Use **Simple Markup** for everyday editing — it's less visually noisy than All Markup. Switch to **All Markup** when you need to review changes carefully.

### Accepting and rejecting changes

When you're done reviewing changes, you can accept or reject them:

- Click **Accept > Accept All Changes** to accept all edits
- Click **Reject > Reject All Changes** to reject all edits
- Click **Accept** or **Reject** individually to accept/reject one change at a time

> [!HEADS-UP]
> Once you accept or reject a change, it's gone — you can't undo it unless you have a backup. Always review changes carefully before accepting them all.

## Comments: feedback without editing

**Comments** let you add feedback to a document without modifying the content. They appear in a bubble on the right side of the page and are linked to the selected text by a colored marker.

Select a word or sentence. On the **Review** tab, click **New Comment**. A comment bubble appears on the right side of the page, and the cursor drops into it. Type your comment.

> [!ASIDE]
> Comments are stored separately from the document content. If you print the document, comments don't appear by default (you can change this in Print Settings). Comments also don't affect the page count.

### Managing comments

On the **Review** tab, click **Next** or **Previous** to jump between comments. The **Show Comments** toggle (in the Comments group) shows or hides all comment bubbles.

> [!PREDICT]
> If you delete the text that a comment is attached to, what happens to the comment? Does it disappear, or does it stay?

## Comparing documents: seeing what changed

Word can compare two versions of a document and produce a third document that shows all the differences. This is useful when you receive a revised document from a reviewer and want to see what they changed.

On the **Review** tab, in the **Compare** group, click **Compare > Compare**. A dialog box opens:

- **Original document** — the original version of your document
- **Revised document** — the revised version
- **Comparing** — choose how to display the comparison (in a new document, or inline)

Click **OK**. Word creates a new document with a **Comparisons** section that shows all the differences between the two versions, using Track Changes markup.

> [!ASIDE]
> Word also has a **Combine** feature (Review > Combine) that merges changes from multiple reviewers into a single document. This is different from Compare: Combine is for merging edits from multiple authors, while Compare is for seeing differences between two specific versions.

## Checkpoint

> [!PREDICT]
> You have a document with a table of contents. You add a new Heading 2 section and change the page numbering. You click "Update page numbers only" in the TOC. What happens?

**Run this to verify your work so far:**

1. Open your saved document from Part 2.
2. Click **View > Navigation Pane** to open the side panel.
3. Click a heading in the Navigation Pane. Verify that Word jumps to that location.
4. Place your cursor at the beginning of the document. Click **References > Table of Contents** and insert the first style. Verify that the TOC appears with dotted tab leaders and page numbers.
5. Add a new Heading 2 section somewhere in the document. Click inside the TOC and click **Update Table > Update entire table**. Verify that the new heading appears in the TOC.
6. Turn on **Track Changes** (Review > Track Changes). Make a few edits (add text, delete text). Verify that insertions are underlined and deletions are struck through.
7. Click **New Comment** on a sentence. Verify that a comment bubble appears on the right side.

**Likely errors:**
- If the TOC doesn't update after adding a new heading, you probably clicked "Update page numbers only" instead of "Update entire table." The former only refreshes page numbers; the latter rebuilds the heading list.
- If Track Changes isn't showing revisions, make sure you're not in **Simple Markup** view — switch to **All Markup** using Display for Review.
- If the comment bubble doesn't appear, make sure you have selected text before clicking New Comment.

## What's next

You now have a document with a table of contents, citations, Track Changes, and comments. But you haven't yet explored the full power of Word's collaborative features: real-time co-authoring on OneDrive, document sharing with permissions, and the Review pane for managing all comments and revisions in one place. The next part will show you how to share your document, collaborate with others in real time, and use Word's review tools to manage feedback at scale.

## Exercises

- [ ] Create a new document with five sections (Heading 1) and three subsections per section (Heading 2). Insert a table of contents, then customize it: remove page numbers, change the tab leader to dashes, and show only Heading 1 and Heading 2 levels.

- [ ] Add three citations to your document using different source types (book, website, journal). Generate a works cited list, then add a fourth source and update the bibliography. Verify that the list is alphabetized and formatted correctly.

- [ ] Turn on Track Changes and make ten edits to your document (five insertions, five deletions). Switch to **No Markup** view, then switch back to **All Markup**. Verify that all changes are visible. Accept three changes and reject three changes.

- [ ] Create two copies of your document. Edit one copy (add text, change formatting). Use **Review > Compare** to compare the two versions. Examine the comparison document and identify all the changes.

- [ ] Open the Navigation Pane. Drag a Heading 2 subsection to a different parent Heading 1. Verify that the content and any sub-subsections move with it. This demonstrates the Navigation Pane's drag-and-drop reordering.

## Sources

1. [Insert a table of contents — Microsoft Support](https://support.microsoft.com/en-us/office/insert-a-table-of-contents-882e8564-0edb-435e-84b5-1d8552ccf0c0) — Official guide on inserting and customizing automatic tables of contents. Used for the TOC insertion and customization steps.

2. [Update a table of contents — Microsoft Support](https://support.office.com/en-us/article/Update-a-table-of-contents-6c727329-d8fd-44fe-83b7-fa7fe3d8ac7a) — Documentation on updating TOCs (page numbers only vs. entire table). Used for the Update Table workflow.

3. [Add or change sources, citations, and bibliographies — Microsoft Support](https://support.microsoft.com/en-us/word/add-or-change-sources-citations-and-bibliographies) — Official guide on Word's citation manager, source types, and bibliography generation. Used for the citation and works cited steps.

4. [Create a bibliography, citations, and references — Microsoft Support](https://support.microsoft.com/en-us/office/create-a-bibliography-citations-and-references-17686589-4824-4940-9c69-342c289fa2a5) — Microsoft's citation style selection and Insert Citation walkthrough. Used for the References tab citation workflow.

5. [Track changes in Word — Microsoft Support](https://support.office.com/en-US/article/Track-changes-in-Word-197ba630-0f5f-4a8e-9a77-3712475e806a) — Official documentation on Track Changes, revision marks, and accepting/rejecting edits. Used for the Track Changes and revision management steps.

6. [Compare and merge two versions of a document — Microsoft Support](https://support.microsoft.com/en-us/office/compare-and-merge-two-versions-of-a-document-f5059749-a797-4db7-a8fb-b3b27eb8b87e) — Guide on using Compare and Combine features. Used for the document comparison workflow.

7. [Combine document revisions — Microsoft Support](https://support.microsoft.com/en-us/office/combine-document-revisions-f8f07f09-4461-4376-b041-89ad67412cfe) — Documentation on merging changes from multiple reviewers. Used for the Combine vs. Compare distinction.

8. [Give and receive feedback in Word — Microsoft Support](https://support.microsoft.com/en-us/office/give-and-receive-feedback-in-word-07f25746-7b26-4769-9ebb-ca2f0f5783b8) — Official guide on comments, sharing, and collaborative editing. Used for the Comments section.

## Closing reflection

Before you move on: in your own words, explain why Word's table of contents is a field rather than static text. Write the answer that would satisfy a skeptical colleague who says "I'll just type the TOC by hand — why bother?" — the answer they need to hear.
