# 🚀 FAQ Master Module  
### **Rich Text FAQ Management | Next.js + PrimeReact + Quill**

<p align="center">
  <img src="https://img.shields.io/badge/Framework-Next.js_14-black?style=for-the-badge&logo=next.js" />
  <img src="https://img.shields.io/badge/UI-PrimeReact-blue?style=for-the-badge&logo=primefaces" />
  <img src="https://img.shields.io/badge/Editor-Quill-green?style=for-the-badge&logo=quill" />
  <img src="https://img.shields.io/badge/Language-TypeScript-blue?style=for-the-badge&logo=typescript" />
</p>

<p align="center">
  <i>A complete FAQ management system with rich text editing, smooth tab navigation, stable dialog behavior, and clean Excel exports.</i>
</p>

---

# 🌟 Overview

The **FAQ Master Module** is a complete solution for creating, editing, storing, and exporting FAQs using rich text content.  
It supports:

- Rich text answers (stored as HTML)
- Editor inside scrollable tabs
- Fully stable editing inside dialogs
- Clean Excel & CSV export (without HTML)
- Zero cursor-jump or re-render issues
- Dynamic editor loading (SSR safe)

---

# 🎯 Features

### 📝 Rich Text Editor
- Implemented using PrimeReact + Quill  
- Dynamic import for SSR safety  
- HTML content storage  

### 🧭 Scrollable Tab Navigation
- Reusable `ScrollableTabView`  
- Auto-hide arrow logic  
- Smooth UX even inside dialogs  

### 📤 Excel & CSV Export
- Export raw text (HTML stripped)  
- Perfect formatting in Excel  

### 💬 FAQ Editing Dialog
- Editor loads with pre-filled HTML  
- No cursor jump  
- No re-render lag  
- No Quill crashes  

---

# 🧱 Tech Stack

| Layer | Technology |
|------|-------------|
| Frontend | Next.js 14 |
| UI | PrimeReact |
| Editor | Quill.js |
| Styling | styled-components |
| Language | TypeScript |
| Backend | Spring Boot |
| Network | Axios |

---

# 🧩 Architecture

```
FAQMaster/
│
├── Add FAQ
│     ├── Question
│     └── Answer (Quill Editor)
│
├── Edit FAQ (Dialog)
│     ├── Scrollable Tab
│     ├── Quill Editor with onLoad patch
│     └── useRef for tracking edits
│
└── FAQ List
      ├── HTML rendered in DataTable
      ├── Edit/Delete actions
      └── Excel/CSV export (raw text)
```

---

# ⚡ Key Implementations

## 1. Dynamic Editor Import (SSR Safe)

```ts
const Editor = dynamic(
  () => import("primereact/editor").then((mod) => mod.Editor),
  { ssr: false }
);
```

---

## 2. Scrollable Tab View Component

Custom component copied and adapted from previous project:

- Styled with `styled-components`
- Scroll arrow auto-hide logic
- Fully reusable in dialogs

---

## 3. Add FAQ – Rich Text Editor

```tsx
<Editor
  value={faqAnswer}
  onTextChange={(e) => setFaqAnswer(e.htmlValue ?? "")}
  style={{ height: "250px" }}
/>
```

---

## 4. Edit FAQ – The Hard Part (Final Architecture)

### Issue:  
Quill re-render → cursor jumps → dialog glitches → HTML not loading.

### Final Fix Strategy:
- Do **NOT** store editor content in React state  
- Use **useRef**  
- Load HTML **only once** via `onLoad`  

### Working Implementation:

```tsx
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
```

### Updating FAQ:

```ts
faq_answer: editAnswerRef.current
```

---

## 5. Display HTML in Table

```tsx
<div
  dangerouslySetInnerHTML={{ __html: row.faq_answer }}
  style={{ whiteSpace: "normal" }}
/>
```

---

## 6. Excel Export – Strip HTML

### Utility:

```ts
const stripHtml = (html) => {
  const temp = document.createElement("div");
  temp.innerHTML = html;
  return temp.textContent || temp.innerText || "";
};
```

### Column Configuration:

```tsx
{
  field: "faq_answer",
  header: "FAQ Answer",
  body: (row) => (
    <div dangerouslySetInnerHTML={{ __html: row.faq_answer }} />
  ),
  exportFunction: (row) => stripHtml(row.faq_answer)
}
```

---

# 🐞 Problems Faced & Solutions

| Problem | Cause | Fix |
|--------|--------|------|
| Cannot resolve styled-components | Missing dependency | Installed styled-components |
| Quill missing | PrimeReact dependency | Install quill |
| Editor defined twice | Duplicate import | Removed static import |
| Editor blank in edit dialog | HTML loaded too early | Use onLoad() |
| Cursor jumps on typing | State update triggers re-render | Switched to useRef |
| Dialog reopening crashes Quill | Old instance reused | Removed useEffect injection |
| TS error: string \| null | Editor returns null | Normalize with `?? ""` |
| Excel shows HTML | Raw HTML exported | stripHtml() utility |

---

# 📸 Screenshots (Add Yours)

```
🖼 Add the following screenshots:
- FAQ List Page
- Add FAQ Page
- Edit FAQ Dialog
- Scrollable Tab UI
- Excel Export Output
```

---

# 🏁 Conclusion

This FAQ Master module is a **fully optimized rich text management solution**, handling:

- Quill lifecycle  
- SSR issues  
- Dialog + TabView re-renders  
- Excel export formatting  
- Clean UI & UX  

It behaves **perfectly** under complex scenarios where Quill normally breaks.

---
