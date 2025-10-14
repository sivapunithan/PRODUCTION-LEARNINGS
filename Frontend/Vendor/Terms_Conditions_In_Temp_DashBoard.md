# 🧾 Vendor Terms & Conditions Enhancement  
### Dynamic “Not Applicable” Rendering (Flag-Based Implementation)

---

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?logo=react&logoColor=black)
![Next.js](https://img.shields.io/badge/Next.js-000000?logo=nextdotjs&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-6DB33F?logo=springboot&logoColor=white)
![Status](https://img.shields.io/badge/Status-Deployed-success?style=flat-square&logo=github)

---

## 🎯 Objective
To make the **Vendor Terms & Conditions** module dynamic by enabling or disabling  
the “Not Applicable (NA)” option based on a backend-controlled field (`flag`).

This ensures:
- ✅ Full backend control over which terms support NA  
- ✅ Dynamic rendering across all forms (create, edit, view)  
- ✅ One unified logic across frontend and backend

---

## 🧩 Files Modified

| Component | Purpose |
|------------|----------|
| `TempVendorDashboard.tsx` | Handles payload generation and parent state management |
| `VendorTermsCondition.tsx` | Dynamic form for vendor responses (create/edit) |
| `VendorTermsConditionDisplay.tsx` | Read-only display of vendor terms and answers |

---

## 🧠 Key Concept – Flag-Based Rendering

The backend now returns a `flag` for each vendor term:

| Flag | Meaning | Behavior |
|------|----------|-----------|
| 0 | Regular term | Only shows “Yes / No” |
| 1 | NA enabled term | Shows “Yes / No / Not Applicable” |

This replaces all hardcoded index-based logic for NA visibility.

---

## ⚙️ Backend Update
Added a new column in the database:

```sql
ALTER TABLE vendor_terms_conditions ADD flag INT NOT NULL DEFAULT 0;
Then, the getTermsByVendorCode API includes this field:

json
Copy code
[
  {
    "id": 1,
    "vendorTerms": "I/We have linked Aadhaar and PAN",
    "vendorAnswer": "na",
    "vendorReason": "-",
    "flag": 1
  },
  {
    "id": 2,
    "vendorTerms": "I/We are compliant with GST Rules",
    "vendorAnswer": "yes",
    "vendorReason": "-",
    "flag": 0
  }
]
🧩 Frontend Implementation
1️⃣ VendorTermsCondition.tsx – Dynamic Form Rendering
tsx
Copy code
{(question.flag === 1 || question.flag === 4) && (
  <label>
    <input
      type="radio"
      name={`question_${question.id}`}
      value="na"
      checked={currentAnswer?.answer === 'na'}
      onChange={() => handleAnswerChange(question.id, 'na')}
    />
    Not Applicable
  </label>
)}
✅ Now, “Not Applicable” is automatically rendered only when flag === 1 or 4.

2️⃣ VendorTermsConditionDisplay.tsx – Dynamic View Mode
tsx
Copy code
const showNaForQuestion = (term: VendorTermsEntity) => term.flag === 1;

{showNaForQuestion(term) && (
  <label>
    <input type="radio" checked={term.vendorAnswer === 'na'} disabled />
    Not Applicable
  </label>
)}
This ensures the read-only vendor view matches the editable form.

3️⃣ TempVendorDashboard.tsx – Payload Update
When saving vendor answers, the flag is now included in the payload:

ts
Copy code
const payload = Object.entries(vendorTermsAnswers).map(([questionId, ans]) => {
  const questionObj = vendorTermsQuestions.find((q) => q.id === Number(questionId));
  return {
    vendorId: Number(storedVendorId),
    vendorCode: storedVendorCode,
    vendorTerms: questionObj?.termsConditions || '-',
    vendorAnswer: ans.answer,
    vendorReason: ans.reason && ans.reason.trim() !== '' ? ans.reason : '-',
    flag: questionObj?.flag ?? 0   // ✅ Include flag
  };
});
🧭 Data Flow Diagram
mermaid
Copy code
flowchart TD
A[VendorTermsCondition] -->|Fetches| B[getvendortermsandconditions API]
B -->|Response includes flag| C[questions[] with flag values]
C --> D{flag === 1?}
D -->|Yes| E[Render Yes / No / Not Applicable]
D -->|No| F[Render Yes / No only]
E --> G[Save response (includes flag)]
F --> G
G --> H[VendorTermsConditionDisplay - View Mode]
🖼️ Example Output
✅ Terms with Flag = 1
yaml
Copy code
1. I/We have linked Aadhaar and PAN
   ( ) Yes   ( ) No   (x) Not Applicable
🚫 Terms with Flag = 0
yaml
Copy code
2. I/We are regular in depositing GST
   (x) Yes   ( ) No
💡 Benefits
Before	After
Hardcoded NA logic	Dynamic, API-driven
No backend control	Controlled per term
Inconsistent across edit/view	Synchronized everywhere
Static payload	Now includes flag metadata