📘 FAQ Master – Full Documentation (Next.js + PrimeReact + Quill)

A fully functional FAQ Management module built using Next.js (App Router), PrimeReact, Quill Editor, and Spring Boot APIs.
This module supports rich-text FAQ creation, editing inside modal dialogs, HTML rendering inside tables, and clean text exports for Excel/CSV.

This README documents the entire development journey, including major problems, root causes, and stable long-term solutions.

🚀 Features
✓ Create / Edit / Delete FAQs

FAQ question

Rich text FAQ answer (stored as HTML)

✓ Rich Text Editor (PrimeReact + Quill)

Dynamic import (SSR-safe)

HTML rendering in table

HTML → raw text conversion for Excel export

✓ Edit FAQ inside Dialog

No re-render issues

No cursor jump

No lost focus

Reusable Scrollable Tab View

✓ Excel / CSV Export

Clean text (no HTML tags)

Multi-line content supported

✓ Reusable ScrollableTabView Component

Smooth tab scroll

Works inside Dialog

Auto-hide arrow behavior

🏗️ Tech Stack

Next.js 14 (App Router)

PrimeReact

Quill.js

styled-components

Spring Boot Backend

Axios

📦 Installation
Install dependencies
npm install primereact primeicons quill styled-components
npm install --save-dev @types/styled-components

Ensure global styles include Quill CSS
import "quill/dist/quill.snow.css";

🧩 Architecture Overview
FAQ Master Page
 ├── DataTable (FAQ List)
 │    ├── Renders HTML Answers
 │    ├── Exports Plain Text to Excel
 │
 ├── Add FAQ Dialog
 │    └── Rich Text Editor inside ScrollableTabView
 │
 └── Edit FAQ Dialog
      ├── Editor initialized via onLoad()
      ├── HTML loaded only once per mount
      ├── Content tracked through useRef()
      └── No unnecessary re-renders

📝 Key Implementation Details
1. Dynamic Import of Quill Editor
const Editor = dynamic(
  () => import("primereact/editor").then((mod) => mod.Editor),
  { ssr: false }
);


Reason: Quill is not SSR-compatible, so dynamic import avoids hydration errors.

2. Scrollable Tab Component

A custom component using styled-components to provide scrollable horizontal tabs.

Supports:

Smooth scroll

Auto hide right arrow

Works inside dialogs

3. Adding FAQ (Create Mode)

FAQ answers stored as HTML:

<Editor
  value={faqAnswer}
  onTextChange={(e) => setFaqAnswer(e.htmlValue ?? "")}
  style={{ height: "250px" }}
/>


htmlValue ?? "" ensures TypeScript safety (string | null → string).

4. Editing FAQ (The Hard Part)
The root problem:

Updating React state on every keystroke causes:

TabView re-render

Dialog re-render

Editor re-render

Cursor jumps to top

Final correct architecture:

Store editor value in a ref

Load answer only once via onLoad

Avoid React state updates during typing

Editor in Edit Dialog:
const editAnswerRef = useRef("");

<Editor
  onLoad={(quill) => {
    quill.setContents([]);
    quill.clipboard.dangerouslyPasteHTML(editFaq.faq_answer);
  }}

  onTextChange={(e) => {
    editAnswerRef.current = e.htmlValue ?? "";
  }}

  style={{ height: "250px" }}
/>

Update API uses ref:
faq_answer: editAnswerRef.current


This ensures:

No cursor jump

No performance issues

No crashes on reopening dialog

5. HTML Rendering Inside DataTable
<div
  dangerouslySetInnerHTML={{ __html: row.faq_answer }}
  style={{ whiteSpace: "normal" }}
/>

6. Excel & CSV Export (Plain Text Only)
Problem:

HTML tags should not appear in Excel exports.

Solution:

Strip HTML before export.

Utility:
const stripHtml = (html: string) => {
  const temp = document.createElement("div");
  temp.innerHTML = html;
  return temp.textContent || temp.innerText || "";
};

Column Definition:
{
  field: "faq_answer",
  header: "FAQ Answer",
  body: (row) => (
    <div dangerouslySetInnerHTML={{ __html: row.faq_answer }} />
  ),
  exportFunction: (row) => stripHtml(row.faq_answer),
}


Now Excel export contains:

hello
hello2

🔧 Major Issues Faced & How They Were Resolved

This is the full battle log of every bug:

1. Missing styled-components

→ Installed missing dependency.

2. Quill not found

→ Installed Quill manually.

3. Editor imported twice

→ Removed static import; kept dynamic import only.

4. Editor showing blank in Edit Dialog

→ Moved HTML loading logic into onLoad callback.

5. Cursor jumps to top while typing

→ Removed setState inside editor; used useRef instead.

6. Quill crashes on reopening dialog

→ Avoided injecting HTML inside useEffect; rely on onLoad.

7. TypeScript error string | null

→ Normalize using e.htmlValue ?? "".

8. Excel/CSV shows HTML

→ Strip HTML before export using DOM method.