# Word for Windows M365: Page Layout, Views, and Structure

> [!RECALL]
> Quick recall before we continue: what three things do you lose if you format every heading manually (bold, size 16, centered) instead of using Heading styles? Name them — you'll see why each matters in this part.

You finished Part 1 with a document that has content — headings, body text, a picture, and a save to OneDrive. But the document has no structure that depends on the page. No margins that aren't the default one inch, no page numbers, no header, no orientation control, and no awareness of what Print Layout, Read Mode, or Web Layout actually do differently. This part fixes that.

By the end of this part, you will have taken your document from a flat stack of text and turned it into a properly paginated report: one-inch margins (or your own custom margins), page numbers in the footer, a header with the document title, and an understanding of when to switch views and why Word gives you three main ones. You will also learn section breaks — the single most important structural concept in Word's page model — because they are the reason you can have different margins, orientations, or page numbering on different pages of the same document.

The version this tutorial targets is **Word for Microsoft 365, version 2605 (build 20026.20140)**. All interface elements referenced are on the Layout tab, Insert tab, or the View tab.

## The Layout tab: margins, size, orientation

Click the **Layout** tab on the ribbon. This tab is where page-level settings live — everything that controls how your content maps to physical or virtual pages. It is organized into three groups: Page Setup (Margins, Orientation, Size, Columns, Breaks), Background (Page Color, Page Borders, Watermark), and Page Setup (again — yes, Word puts some page setup controls in two places).

> [!HEADS-UP]
> Don't let the duplicated groups confuse you. The Page Setup group on the left controls margins, orientation, size, columns, and breaks. The Page Setup group on the right has a small dialog launcher (a tiny arrow in the bottom-right corner of the group) that opens the full Page Setup dialog box with tabs for Margins, Paper, Layout, and Document Properties. Use the ribbon for quick changes; use the dialog box for precision.

### Margins

Your document currently has the default margins: **one inch** on all four sides (top, bottom, left, right). Word applies this automatically when you create a new document.

To change them, click **Margins** in the Page Setup group. A dropdown appears with six predefined options:

- **Normal** — 1" top, 1" bottom, 0.75" left and right
- **Narrow** — 0.5" on all sides
- **Wide** — 1" top/bottom, 1.25" left/right
- **Gutter** — adds extra space along the left margin for binding (useful for reports you plan to bind)
- **Mirror margins** — asymmetric left/right margins for booklet or double-sided printing (inner margins differ from outer)
- **Custom Margins** — open the full dialog to set exact values

> [!PREDICT]
> Why do you think "Mirror margins" exists? What real-world printing situation would benefit from asymmetric margins?

Click **Margins > Narrow**. Your page shrinks — the text area grows because the margins shrank from 1" to 0.5". Notice that the change applies to the entire document. If you had multiple sections later, you could set different margins per section (see "Section breaks" below).

> [!ASIDE]
> "Narrow" margins are 0.5", not 0.25". Word's smallest predefined margin is half an inch. If you need tighter than that, use Custom Margins.

### Page size

Click **Size** in the Page Setup group. The default is **Letter** (8.5 × 11 inches, or 216 × 279 mm). The dropdown shows: Letter, Legal (8.5 × 14 inches), Tabloid (11 × 17 inches), A4, A5, Executive, and a few more.

> [!ASIDE]
> "Letter" is a North American standard. If you're in Europe, Asia, or most of the world, your default is likely A4. Word adapts to your system locale. You can change it here regardless.

Click **Size > A4** (210 × 297 mm). The document reflows slightly — the page is taller and narrower than Letter. The content reflows to the new page boundaries automatically.

### Orientation

Click **Orientation** in the Page Setup group. You'll see two options: **Portrait** and **Landscape**.

Portrait is the default — the page is taller than it is wide. Landscape flips the page 90 degrees — it is wider than it is tall.

Click **Orientation > Landscape**. Your page rotates. The text reflows to the new width. If you had a wide table, this might be exactly what you need.

> [!TIP]
> To set orientation for just one page while keeping the rest in Portrait, you'll use a section break. That's coming up in the "Section breaks" section below.

## The three views: Print Layout, Read Mode, Web Layout

At the bottom-right of the Word window, on the Status Bar, you'll find three view buttons. From left to right: **Print Layout** (the default), **Read Mode**, and **Web Layout**. You can also access these from the **View** tab on the ribbon.

### Print Layout — the working view

Print Layout is the default view and the one you will use most often. It displays your document exactly as it will appear when printed: with margins, page breaks, headers, footers, and all other page-level elements visible.

> [!PREDICT]
> What do you think a "page break" looks like in Print Layout? Is it just a visual divider, or does it affect how content flows?

In Print Layout, page breaks appear as thick, dark horizontal bars between pages. Content flows from one page to the next automatically. If you type until you reach the bottom of the page, Word creates a new page — you don't need to insert page breaks manually (though you can, using **Layout > Breaks > Page Break**).

### Read Mode — the reading view

Click **Read Mode** (or the second view button on the Status Bar). The window switches to a full-screen reading experience. Pages are displayed as if they were stacked on a table, and you flip between them by clicking or using arrow keys.

Read Mode hides the ribbon, the Quick Access Toolbar, and most of the UI chrome. It's optimized for reading, not editing. The page is centered on the screen with a slight shadow, and adjacent pages are visible in the background — so you can see context without leaving the current page.

> [!ASIDE]
> Read Mode is the digital equivalent of picking up a printed document and holding it in your hands. If you're reviewing a document rather than writing it, switch to Read Mode. It's easier on the eyes.

### Web Layout — the flow view

Click **Web Layout** (or the third view button on the Status Bar). The document now looks like a web page: no page breaks, no margins (or minimal margins), and content flows continuously from top to bottom as if you were scrolling through a webpage.

Web Layout is useful for creating documents that will be viewed digitally — email attachments, web pages, or documents shared as PDFs where the reader is unlikely to print. It removes the "page" concept entirely, so you see the document as a continuous flow.

> [!HEADS-UP]
> In Web Layout, headers, footers, and page numbers are not visible. They exist in the document but are hidden in this view. If you've added page numbers and they don't show up in Web Layout, that's normal — they only render in Print Layout and Read Mode.

> [!DESIGN-NOTE]
> **Why three views?** Word is both a word processor and a typesetting engine. Print Layout bridges the two: it shows you what you're writing *and* what it will look like when printed. Read Mode is pure consumption — no editing chrome, just pages. Web Layout is pure flow — no page constraints at all. Each view answers a different question:
>
> - Print Layout: "How will this look when printed?"
> - Read Mode: "How does this read on screen?"
> - Web Layout: "How does this flow without page breaks?"
>
> The reason Word gives you all three is that the answer to each question is different, and none of them is the same as what the other shows.

## Headers and footers: the invisible layer

Headers and footers live in a separate layer from the main document body. They appear at the top and bottom of every page, respectively, and they repeat automatically on every page. You don't type them into the document — you insert them into the header or footer area, and Word handles the rest.

On the **Insert** tab, in the **Header & Footer** group, click **Header**. A dropdown appears with several pre-designed header layouts. Click the first one (a simple blank header) or any style you prefer.

> [!RECALL]
> Quick recall: what happens when you format a heading manually vs. using a Style? You'll see the answer to this question in action when you learn about section breaks.

The cursor drops into the header area, the main document body dims (you're now editing the header), and a new contextual tab appears on the ribbon: **Header & Footer Design**. Type something — for example, your document title "My First Word Document" — and press **Tab** to move to the right side. Type the date or your name.

Now click **Close Header and Footer** (or double-click in the main body). The header is saved and appears at the top of every page in Print Layout.

Click **Insert > Footer** and add a page number. Choose **Bottom of Page > Plain Number 2** (centered). The page number appears at the bottom center of every page.

> [!ASIDE]
> Page numbers in Word are automatic fields, not typed numbers. Word counts the pages and inserts the number dynamically. If you add or remove content and pages shift, the page numbers update automatically.

## Section breaks: the master key to page-level control

This is the most important concept in this part, and it's the one that separates Word users from Word *knowers*.

A **section break** divides a document into independent sections. Each section can have its own margins, orientation, page size, header/footer, and page numbering. Without section breaks, the entire document is one section — and changing a page-level setting affects every page.

> [!PREDICT]
> You have a 10-page document in Portrait orientation. You want pages 4–6 to be in Landscape (for a wide table), and the rest in Portrait. Without section breaks, you can't do this. Why not? What would happen if you just clicked Landscape on page 5?

Click at the beginning of page 4 (before the content on that page). On the **Layout** tab, click **Breaks > Page Break** (under the Page Breaks section). No, wait — that's not a section break. Click **Breaks > Section Break: Next Page** instead.

Now your document has two sections: Section 1 (pages 1–3) and Section 2 (pages 4–10). The section break is invisible in normal editing, but it's there — and it gives you power.

With the cursor in Section 2, click **Orientation > Landscape**. Only pages 4–10 rotate. Pages 1–3 stay in Portrait. This is what section breaks do: they let you change page-level settings for *part* of the document without affecting the rest.

> [!HEADS-UP]
> Section breaks are invisible by default. If you can't see them, go to the **Home** tab and click the **¶** (Show/Hide) button. This reveals formatting marks, including section breaks (they appear as a double-dashed line labeled "Section Break (Next Page)"). You'll also see paragraph marks (¶) and page breaks (a single dashed line).

> [!ASIDE]
> Word has four types of section breaks:
> - **Next Page** — starts the new section on the next page (what you just used)
> - **Continuous** — starts the new section on the same page (useful for changing columns mid-page)
> - **Even Page** — starts the new section on the next even-numbered page
> - **Odd Page** — starts the new section on the next odd-numbered page
>
> The last two are used for book layouts. You'll rarely need them, but they're there if you do.

## Different First Page: headers and footers per section

Click back into the header area (double-click at the top of page 1). On the **Header & Footer Design** tab, you'll see a checkbox called **Different First Page**. Check it.

The header on page 1 changes: the text you typed earlier disappears from page 1's header but remains on pages 2–10. This is because page 1 now has its own header, separate from the headers on the remaining pages.

> [!PREDICT]
> If Different First Page is checked, what do you think happens if you type new text into page 1's header? Will it also appear on page 2?

Type something new into page 1's header — your document's title, perhaps. Page 2's header retains the original text. They are now independent because Different First Page created a separate header for the first page.

> [!ASIDE]
> Different First Page is why cover pages don't need page numbers. The first page's header/footer is separate from the rest, so you can omit the page number on the cover while keeping it on every other page.

## Checkpoint

> [!PREDICT]
> You have a document in Portrait with one-inch margins. You insert a Section Break: Next Page after page 3, then switch to Landscape orientation. Then you insert another Section Break: Next Page after page 6, and switch back to Portrait. How many sections does your document have, and what are their orientations?

**Run this to verify your work so far:**

1. Open your saved document from Part 1.
2. Click the **Layout** tab, then **Margins > Narrow**. Verify the page text area grows.
3. Click **Orientation > Landscape**. Verify the page rotates.
4. Click **Orientation > Portrait** to restore it.
5. Click **Margins > Normal** to restore one-inch margins.
6. Click **View > Print Layout** (or the first view button on the Status Bar). Verify you see page breaks as dark horizontal bars.
7. Click **View > Read Mode** (or the second view button). Flip through a few pages, then press **Esc** to return to Print Layout.

**Likely errors:**
- If the orientation change affects all pages instead of just part of the document, you probably forgot to insert a section break first. Without a section break, the entire document is one section, and orientation changes apply globally.
- If page numbers don't show up in Web Layout, that's expected — they only appear in Print Layout and Read Mode.
- If the header text appears on every page including page 1, you probably didn't check **Different First Page** in the Header & Footer Design tab.

## What's next

You now have a document with proper margins, headers, footers, page numbers, and section breaks. But you haven't yet explored the document's structural tools: the Table of Contents (which reads your heading hierarchy and builds itself), the Index, Citations, and the Review tools (Track Changes, Comments, Compare). The next part will add a Table of Contents to your report and show you how Word's reference tools work — because a document with headings is one thing, but a document with an auto-generated table of contents is something else entirely.

## Exercises

- [ ] Create a new section break after page 2 (Layout > Breaks > Section Break: Next Page). Change the orientation of Section 2 to Landscape. Insert a page number in the footer of Section 2 only, then change the section break type to Section Break: Continuous. Observe how the orientation change behaves differently with a continuous break.

- [ ] In your existing document, add a header with your name on the left and the document title on the right (use Tab to position). Enable Different First Page and type a unique header for page 1. Save the document and switch to Print Layout — verify that page 1 has a different header than the rest.

- [ ] Switch your document to Web Layout. Type a long paragraph until it fills more than one "page" (remember, Web Layout has no page breaks). Now switch back to Print Layout. Notice how the content reflows into pages. Estimate how many pages your document now has.

- [ ] Open a new blank document. Type five lines of text. Format the first as Heading 1, the second as Heading 2, the third as Heading 2, the fourth as Heading 1, and the fifth as Heading 3. Switch to Read Mode. Observe how the pages are formatted as columns — Read Mode doesn't just show one page at a time; it reflows the content into a newspaper-style layout.

## Sources

1. [Change margins — Microsoft Support](https://support.microsoft.com/en-us/office/change-margins-da21a474-99d8-4e54-b12d-a8a14ea7ce02) — Official documentation on margin presets and custom margins. Used for the Margins dropdown walkthrough and the 0.5" / 0.75" / 1" values.

2. [Add page numbers to a header or footer in Word — Microsoft Support](https://support.microsoft.com/en-us/office/add-page-numbers-to-a-header-or-footer-in-word-46d6dfe5-f99b-40d8-8809-be4808a291f4) — Official guide on inserting and customizing page numbers. Used for the footer and page number positioning steps.

3. [Insert page numbers — Microsoft Support](https://support.office.com/en-us/article/insert-page-numbers-9f366518-0500-4b45-903d-987d3827c007) — Microsoft's page number insertion reference. Used for the Bottom of Page > Plain Number 2 selection.

4. [Exploring Different Document Views in Microsoft Word — Noble Desktop](https://blog.nobledesktop.com/learn/microsoft-word/exploring-different-document-views-in-microsoft-word) — Detailed comparison of Print Layout, Read Mode, Web Layout, Outline, and Draft views. Used for the view descriptions.

5. [How to work with different views in Microsoft Word — TechRepublic](https://www.techrepublic.com/article/how-to-work-with-different-views-in-microsoft-word/) — Practical guide to switching between views and understanding when to use each. Used for the Read Mode and Web Layout explanations.

6. [How to Add Page Numbers in Word – Starting on a Specific Page — Excel at Work](https://www.excelatwork.co.nz/2026/03/29/how-to-add-page-numbers-in-word-starting-on-a-specific-page/) — Detailed explanation of page numbers as automatic fields, Different First Page, and section breaks. Used for the section break and header/footer independence concepts.

7. [How to Insert a Header and Footer in Microsoft Word — TheLinuxCode](https://thelinuxcode.com/how-to-insert-a-header-and-footer-in-microsoft-word-practical-section-safe-and-hard-to-break/) — Practical guide to header/footer insertion and section-safe editing. Used for the Header & Footer Design tab workflow.

## Closing reflection

Before you move on: in your own words, explain why a section break is more important than a page break. Write the answer that would satisfy a skeptical colleague who says "I'll just press Enter until the landscape page starts" — the answer they need to hear.
