# Word for Windows M365: Your First Document

You stare at a blank white page. It looks simple — a cursor blinking like a metronome, waiting for you to do something. But Word for Microsoft 365 is not a notepad. It is a full-featured typesetting engine with a ribbon full of tabs, styles that can restructure your document, and a cloud sync system that means your file lives everywhere. The gap between "I can type and save" and "I actually know Word" is wider than the gap between any two versions.

This tutorial closes that gap. You will go from zero to a properly structured document — with headings, a table of contents, page numbers, and formatted text — all within Word for Microsoft 365 on Windows. Along the way, you will meet the ribbon, the Quick Access Toolbar, the Styles gallery, the File menu, and the Copilot button (if your subscription includes it). You will learn how Word's document model differs from what you might expect from a simpler editor, and why that difference matters when you need to produce something that looks professional.

The version this tutorial targets is **Word for Microsoft 365, version 2605 (build 20026.20140)**, released June 2, 2026. This version includes the Fluent Design visual refresh and the Copilot Dynamic Action Button. If your build is slightly older, the interface will be recognizably the same — the core concepts do not change between monthly updates.

> [!HEADS-UP]
> Word for Microsoft 365 updates monthly. Menus shift very slowly over time, but the core layout has been stable since Word 2013. If you see a button in a slightly different spot, it is not a bug — you are looking at a different build.

## What you'll build

By the end of this part, you will have created a two-page document from scratch — a short report with a title, three heading levels, formatted body text, a bulleted list, and a picture. You will save it to OneDrive, close Word, reopen it, and verify everything survived the round-trip.

## Prerequisites

- **Windows 10 or Windows 11** — this tutorial assumes Win32 Word, not the web app or the Mac version.
- **A Microsoft 365 subscription** — any tier (Personal, Family, Business) that includes Word for Windows. The standalone perpetual versions (Word 2021, Word 2024) look nearly identical but lack Copilot and some cloud features.
- **A Microsoft account** signed into Word — this is how your subscription is activated and how OneDrive sync works.
- **No prior Word experience required.** If you have used a basic text editor before, that is enough.

## Starting Word and reading the window

The first thing to understand about Word is that it does not open directly to a blank page. It opens to the **Start screen** — a landing area where you choose what kind of document to create. This is by design: Word wants you to know about templates before you commit to a blank page.

To launch Word, click the **Start** button (or press the Windows key) and type `Word`. Click the app icon. The Start screen appears.

> [!PREDICT]
> Before you click anything: the Start screen shows a search box at the top that says "Search for online templates." What do you think happens if you type "report" there and press Enter?

At the top of the window, across the full width, sits the **Title Bar**. It shows the document name ("Document1" by default) and, on the far right, the standard Minimize, Restore, and Close buttons. Just below that is the **Quick Access Toolbar** — a small horizontal strip with three icons by default: **Save** (a floppy disk), **Undo**, and **Redo**. You can customize this toolbar by clicking the small downward arrow at its right end.

Below the Quick Access Toolbar is the **File tab** — the large, dark-colored button on the far left of the ribbon strip. This is not a tab you click to work on your document. The File tab opens **Backstage View**, a full-screen menu for document-level operations: New, Open, Save, Save As, Print, Share, Export, Close, and Account. You will use it to save and open files, but never to format text.

The ribbon itself sits below the File tab and Quick Access Toolbar. It is organized into **tabs** (Home, Insert, Design, Layout, References, Review, View, and sometimes others like Help or Copilot). Each tab contains **groups** of related commands. The groups change when you switch tabs, and some groups even change when you select something — for example, selecting a table reveals a **Table Design** and **Layout** tab that did not exist before. This is called **contextual tabs**, and it is one of the things that makes Word feel crowded at first. Every command lives somewhere, but you have to know which tab to check.

The large white area in the center is the **Edit Window** — your document canvas. Below it, the **Status Bar** shows the current page number, word count, and a few other indicators. At the bottom-right corner of the window, you will find the **Zoom slider** and view buttons (Print Layout, Read Mode, Web Layout).

> [!ASIDE]
> The "ribbon" was introduced in Word 2007 and replaced the old menu-bar-and-toolbar model. If someone over 40 keeps saying "remember when you had to click File, then New, then OK?" — that is what they are nostalgic for.

## Creating your first document

Back on the Start screen, you will see several options. The top-left option is **Blank document** — a plain white page. Below that are template thumbnails: Invoice, Resume, Report, Newsletter, and dozens more. The search box at the top lets you find online templates by keyword.

Click **Blank document**. Word opens a new document named "Document1" and places the cursor at the top-left of the page, ready for input.

> [!TIP]
> You can also create a blank document from within an open document by clicking the **File** tab, then **New**, then **Blank document**. Or press **Ctrl + N** — the keyboard shortcut works from anywhere.

## The ribbon: Home and the Font group

Click the **Home** tab. This is the default tab — it opens automatically when you launch Word and it is where you spend 80% of your time. The Home tab is organized into several groups: Clipboard, Font, Paragraph, Styles, Editing.

Let's work with the **Font group**. Type the following text:

```
My First Word Document
```

Now select it by clicking at the start, holding the mouse button, and dragging to the end. The selected text will be highlighted with a dark background.

Look at the Font group. You will see buttons for **Bold** (**B**), **Italic** (**I**), **Underline** (**U**), **Strikethrough**, **Font** name, **Font Size**, and a few more. Click **Bold**. Your selected text is now bold. Click **Bold** again — it toggles off. This is how most Word buttons work: click once to apply, click again to remove.

Now change the font size to **16** by clicking the font size dropdown and selecting 16 (or typing 16 and pressing Enter). The text is now larger and bold.

> [!ASIDE]
> "Font" and "typeface" are often used interchangeably, but in Word's terminology, **font** is the correct word. A typeface is the design (Calibri, Times New Roman); a font is a specific size and weight of that typeface (Calibri Bold 16pt). Word uses "font" everywhere.

## The paragraph: alignment and spacing

Select your title text again. On the **Home** tab, look at the **Paragraph group**. You will see alignment buttons (Left, Center, Right, Justify), a **Bullets** button, a **Numbering** button, and line spacing controls.

Click **Center**. Your title is now centered on the page.

> [!PREDICT]
> The line spacing dropdown shows options like 1.0, 1.15, 1.5, and 2.0. What does 1.0 mean? Is it the same as "single spacing"?

The default line spacing in Word for M365 is **1.08** — not 1.0, not 1.15. It is a specific number that changed from the older 1.08 default in a previous update. You can verify this by clicking the line spacing dropdown and looking at the currently selected value.

## Using Styles: the right way to format headings

Here is where Word diverges from simpler editors. You *could* format your title by making it bold and size 16. But Word has a better system: **Styles**.

A Style is a named collection of formatting rules — font, size, color, spacing, indentation — bundled into a single click. Word ships with dozens of built-in styles, including **Title**, **Heading 1**, **Heading 2**, **Heading 3**, **Normal**, **Quote**, and more.

> [!HEADS-UP]
> If you format every heading manually (bold, size 16, centered), you will lose three things: automatic table-of-contents generation, consistent formatting across your document, and the ability to re-skin your entire document with one click via Style Sets. Use Styles. Always.

Your title text is already bold and size 16. Now click the **Styles** group on the Home tab. You will see a row of style thumbnails. Hover over **Heading 1** — you will see a live preview of your selected text formatted as a heading, without actually applying it yet. Click **Heading 1**. Your title is now formatted using the Heading 1 style, which applies a specific font, size, color, and spacing — all defined by the current Style Set.

> [!RECALL]
> Quick recall: what happens if you hover over a style in the Styles gallery without clicking it?

Now type a second line below your title:

```
Getting Started with Word
```

Press **Enter** to move to a new line. Select this text and click **Heading 2** in the Styles gallery. This is your first subheading.

Type a third line:

```
A Beginner's Guide
```

Select it and click **Heading 3**. You now have a three-level heading hierarchy — the kind of structure that Word's table-of-contents engine can read.

## Adding body text and basic formatting

Press **Enter** twice to create some space. Now type a paragraph of body text:

```
Microsoft Word for Microsoft 365 is a word processing application that is part of the Microsoft 365 subscription. Unlike the one-time purchase versions (Word 2021, Word 2024), the M365 version receives continuous feature updates through the Current Channel, Monthly Enterprise Channel, or Semi-Annual Enterprise Channel, depending on your organization's update policy.
```

With the text still selected, click **Normal** in the Styles gallery. This applies the default body style — typically Calibri 11pt in the current Style Set. The Normal style is the foundation of every Word document; every paragraph that does not have an explicit style assigned inherits from Normal.

Now select the phrase "Microsoft 365" in that paragraph and click **Italic**. Select "Current Channel" and click **Bold**. Select "Monthly Enterprise Channel" and click **Underline**. You have now mixed formatted runs inside a single paragraph — Word stores each formatting change as a separate "run" within the paragraph.

> [!ASIDE]
> "Run" in Word's document model (the Office Open XML format) is a contiguous sequence of text with identical formatting. A paragraph with 500 words where only three words are bold still contains five runs: unformatted, bold, unformatted, bold, unformatted. This is why Word files can get large — every formatting change creates a new run element.

## Inserting a picture

Click the **Insert** tab. This tab contains everything you add *into* your document that is not plain text: pictures, shapes, tables, charts, icons, SmartArt, headers and footers, page numbers, and more.

In the **Illustrations** group, click **Pictures**. You will see options for **This Device** (select a file from your computer) and **Online Pictures** (search the web). Click **This Device** and navigate to any image file — a photo, an icon, anything. Click **Insert**.

The picture appears in your document at the current cursor position. By default, Word places it on a **In Line with Text** wrapping mode, meaning it behaves like a very tall character in your paragraph. You can change this by clicking the picture and using the **Layout Options** button that appears next to it — switch to **Square** or **Tight** wrapping if you want text to flow around the image.

> [!TIP]
> Resize any picture by dragging one of its corner handles. Always use a corner handle, never a side handle — side handles distort the aspect ratio.

## Saving to OneDrive

Click the **File** tab to enter Backstage View. On the left sidebar, click **Save As**. You will see two main options:

- **OneDrive** — saves to your cloud storage, enabling auto-save and real-time co-authoring
- **This PC** — saves to your local hard drive

For this exercise, click **OneDrive** and then **Browse**. If you are not signed in, you will be prompted to sign in with your Microsoft account. Choose a folder, type a file name (e.g., `MyFirstDocument`), and click **Save**.

> [!HEADS-UP]
> If you save to OneDrive, Word enables **AutoSave** — a small toggle switch in the top-left corner of the window, next to the Undo button. When AutoSave is on, your document is saved continuously to OneDrive as you type. If you save to This PC instead, AutoSave will be off (or grayed out). Know which you are using.

Click the **X** in the upper-right corner of the window to close Word. If you had unsaved changes, Word will ask if you want to save them. Click **Yes**.

## Reopening and verifying

Open Word again. On the Start screen, under **Recent**, you will see `MyFirstDocument` listed. Click it. The document opens exactly where you left it — same cursor position, same content, same formatting. This is the round-trip test: your document survived save, close, and reopen.

> [!PREDICT]
> You have a document with three heading levels and body text. What do you think happens if you click the **References** tab and then **Table of Contents**?

## Checkpoint

> [!PREDICT]
> Before you run this: you have formatted text with Bold, Italic, and Underline applied to different parts of the same paragraph. If you select the entire paragraph and click **Clear All Formatting** in the Font group, what happens?

**Run this to verify your work so far:**

1. Open your saved document in Word.
2. Select the entire first body paragraph (the one about Microsoft 365).
3. Click **Clear All Formatting** (the eraser icon in the Font group).
4. Observe the result: all bold, italic, and underline formatting is removed, and the text reverts to the Normal style.
5. Click **Undo** (the curved arrow in the Quick Access Toolbar) to restore the formatting.

**Likely errors:**
- If the **Clear All Formatting** button is grayed out, you probably have no selection — click somewhere in the paragraph first.
- If the document does not reopen from **Recent**, you may have saved it to **This PC** instead of OneDrive. Open Word, click **File > Open > Browse**, and navigate to where you saved it.

## What's next

You have a document with headings, body text, a picture, and you have saved it to OneDrive. But you have not yet explored page layout (margins, orientation, columns), headers and footers, page numbers, or the document's structural tools like a table of contents. The next part will add those elements and show you how Word's page model works — margins, page size, orientation, and the difference between Print Layout and Read Mode.

## Exercises

- [ ] Create a new blank document. Type three lines of text. Format the first line as Heading 1, the second as Heading 2, and the third as Heading 3. Then change the entire document's style set by clicking the **Design** tab and selecting a different Style Set from the **Document Formatting** gallery. Observe how all your headings change appearance at once.

- [ ] Insert a bulleted list: type five items, each on its own line. Select them all, then click the **Bullets** button in the Paragraph group. Now press **Enter** between two items, type a new item, and press **Tab** before typing it — observe how the indentation changes the bullet level.

- [ ] Save the document to This PC (not OneDrive) as `Exercise3.docx`. Close Word. Reopen Word and open the file from File Explorer (navigate to where you saved it). Compare this to reopening from the Recent list — notice the difference in steps required.

- [ ] Open the document you created in this tutorial. Change the font size of all Heading 1 text from the default to 18pt by right-clicking the **Heading 1** style in the Styles gallery and selecting **Modify**. Change the font to Calibri, size 18, bold, and click OK. Observe how every Heading 1 in your document updates automatically.

## Sources

1. [Word for new users — Microsoft Support](https://support.microsoft.com/en-au/office/word-for-new-users-cace0fd8-eed9-4aa2-b3c6-07d39895886c) — Official Microsoft guide covering the Word interface, saving, formatting, and styles. Used for the UI tour and basic formatting steps.

2. [Create a document in Word — Microsoft Support](https://support.microsoft.com/en-us/word/training/create-a-document-in-word) — Microsoft's official training article on creating documents, adding text, and inserting pictures. Used for the Insert tab walkthrough.

3. [The new look of Office — Microsoft Support](https://support.microsoft.com/en-us/office/the-new-look-of-office-a6cdf19a-b2bd-4be1-9515-d74a37aa59bf) — Documentation on the Fluent Design visual refresh applied to Office apps in 2025–2026. Used for the theme and visual update context.

4. [Update history for Microsoft 365 Apps — Microsoft Learn](https://learn.microsoft.com/en-us/officeupdates/update-history-microsoft365-apps-by-date) — Official version and build history. Used to pin the target version (2605, build 20026.20140).

5. [The Copilot Dynamic Action Button in Word, Excel, and PowerPoint — Microsoft Support](https://support.microsoft.com/en-gb/topic/the-copilot-dynamic-action-button-in-word-excel-and-powerpoint-40db4cef-3d59-474d-9dec-f649b5bfab8e) — Documentation on the new Copilot entry point rolled out in 2026. Used for Copilot button context.

6. [What's the difference between Microsoft 365 and Office 2021? — Microsoft Support](https://support.microsoft.com/en-us/office/what-s-the-difference-between-microsoft-365-and-office-2021-ed447ebf-6060-46f9-9e90-a239bd27eb96) — Comparison of subscription vs. perpetual licensing models. Used to clarify M365 vs. 2021 differences.

## Closing reflection

Before you move on: in your own words, explain why Word's Style system is more powerful than manual formatting. Write the answer that would satisfy a skeptical colleague who says "I'll just make my headings bold and big — why do I need Styles?" — the answer they need to hear.
