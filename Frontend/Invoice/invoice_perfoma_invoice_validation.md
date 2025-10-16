# 🧾 Documentation — **Processing Qty Validation + Performa Sync**

**Author:** Shivapunithan S  
**Date:** 2025-10-16  
**Module:** Invoice Management → Upload Invoice  
**Tables Involved:**  
- `Performa_Invoice_Assets` (Parent)  
- `Upload_Invoice_Assets` (Child)  

---

## 🎯 **Goal**

To prevent data inconsistency between **Performa Invoice Assets (Parent)** and **Upload Invoice Assets (Child)** by:
1. Fetching **live remaining quantities** from Performa invoice for validation.  
2. Performing frontend validation before user updates the Processing Quantity (`processingQty`).  
3. Updating backend values correctly when quantity is increased or decreased.  

---

## ⚙️ **Problem Background**

In our workflow, we have **three main pages**:
1. **Performa Invoice** → Source (Parent data)  
2. **Upload Invoice** → Child (records created based on performa)  
3. **Invoice Validation / Confirmation** → Final review & modification  

---

### 🧩 The Relationship

- Each **Upload Invoice Asset** record is created **from** a corresponding **Performa Invoice Asset**.  
- During this creation, we store the **`invAstId`** (Primary Key of Performa Invoice Assets) inside Upload Invoice Assets.  
- This means each invoice asset line knows which Performa asset it originally came from.

---

### ⚠️ The Real Problem

When multiple invoices are created for the same PO:
- The **Performa Invoice Assets (parent)** keeps getting updated (its `qtyProcessedNew` changes).  
- But the **Upload Invoice Assets (child)** has no live link — it only remembers the *initial* data copied during its creation.  
- As a result, even if parent performa data changes, the existing upload invoice asset rows don’t reflect it.

So, validations like:
> “Don’t exceed available quantity”

…were failing because the frontend only had stale data.

---

## 🚀 **The Fix**

### 1️⃣ **Created a New API**
Purpose: To fetch the **live Remaining Qty** from `Performa_Invoice_Assets`.

This API returns the *remaining quantity available* for each item under a PO:
```js
GET /api/v1/getRemainingQtyByPo/{poId}
```

Used in frontend like:
```tsx
useEffect(() => {
  const fetchRemainingQty = async () => {
    const apiUrl = await getUrl();
    const token = sessionStorage.getItem('token');
    const res = await axios.get(`${apiUrl}api/v1/getRemainingQtyByPo/${poId}`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    setRemainingData(res.data || []);
  };

  if (poId) fetchRemainingQty();
}, [poId]);
```

This ensures **validation always uses latest data from DB**, not old copied data.

---

### 2️⃣ **Frontend Validation Logic**

#### File: `UploadInvoicePage.tsx`

When user changes Processing Quantity in table →  
The function `handleEditChange()` triggers and validates the value.

```tsx
const handleEditChange = (field, uiAssetId, value) => {
  setPoColumnData((prev) =>
    prev.map((row) => {
      if (row.uiAssetId !== Number(uiAssetId)) return row;
      const next = { ...row, [field]: value };

      const rawInput = Number(value);
      if (isNaN(rawInput) || rawInput < 0) {
        toast.current.display('warn', 'Processing Quantity must be a positive number.');
        return row;
      }

      // Fetch live performa record
      const liveRecord = remainingData.find(
        (item) => Number(item.invAstId) === Number(row.invAstId)
      );

      if (liveRecord) {
        const totalQty = Number(liveRecord.totalQty || 0);
        const processedNew = Number(liveRecord.qtyProcessedNew || 0);
        const performaRemaining = Math.max(0, totalQty - processedNew);
        const maxAllowed = row.receivedQty + performaRemaining;

        if (rawInput > maxAllowed) {
          toast.current.display(
            'warn',
            `Max allowed processing quantity is ${maxAllowed}. You can only add ${performaRemaining} more unit(s).`
          );
          return row;
        }
      }

      next.processingQty = rawInput;
      next.totalPrice = +(rawInput * row.rate).toFixed(2);
      return next;
    })
  );
};
```

✅ **Frontend now ensures**:
- User can’t type more than remaining qty.  
- If backend data changed recently, the latest available qty is used.  
- Warns user with a clear message if limit exceeded.  

---

### 3️⃣ **Backend Sync Logic**

#### File: `UploadInvoiceNewService.java`

```java
@Transactional
public List<UploadInvoiceAssetsEntity> updateOrInsertAssets(int invoiceId, List<UploadInvoiceAssetsEntity> assetsEntity) {
    List<UploadInvoiceAssetsEntity> result = new ArrayList<>();

    UploadInvoiceEntity invoiceEntity = uploadInvoiceRepository.findById(invoiceId)
        .orElseThrow(() -> new RuntimeException("UploadInvoice not found for id: " + invoiceId));
    String poNonPo = invoiceEntity.getPoNonPo();

    for (UploadInvoiceAssetsEntity row : assetsEntity) {
        UploadInvoiceAssetsEntity entity = row.getUiAssetId() != null
            && uploadInvoiceAssetsRepository.existsById(row.getUiAssetId())
            ? uploadInvoiceAssetsRepository.findById(row.getUiAssetId()).get()
            : new UploadInvoiceAssetsEntity();

        entity.setInvoiceId(invoiceId);

        // Fetch old record to compare
        UploadInvoiceAssetsEntity oldAsset = uploadInvoiceAssetsRepository.findById(row.getUiAssetId())
            .orElseThrow(() -> new RuntimeException("UploadInvoice asset not found for id " + row.getUiAssetId()));

        int oldProcessingQty = oldAsset.getReceivedQty();
        int newProcessingQty = row.getReceivedQty();

        // copy all fields
        entity.setReceivedQty(newProcessingQty);
        entity.setRate(row.getRate());
        entity.setTotalPrice(row.getTotalPrice());
        ...
```

---

### 🔹 **PO Flow — Update Performa Invoice Assets**

```java
if ("PO".equalsIgnoreCase(poNonPo)) {
    List<PerformaInvoiceAssetsEntity> perAssetsList =
        performaInvoiceAssetsRepo.findAllByInvAstId(row.getInvAstId());
    if (!perAssetsList.isEmpty()) {
        PerformaInvoiceAssetsEntity perfAsset = perAssetsList.get(0);
        int totalQty = perfAsset.getQty();
        int processedSoFar = perfAsset.getQtyProcessedNew();

        int updatedProcessed;
        if (newProcessingQty > oldProcessingQty) {
            updatedProcessed = processedSoFar + (newProcessingQty - oldProcessingQty);
        } else if (newProcessingQty < oldProcessingQty) {
            updatedProcessed = processedSoFar - (oldProcessingQty - newProcessingQty);
        } else {
            updatedProcessed = processedSoFar;
        }

        // Cap the value between 0 and total
        updatedProcessed = Math.max(0, Math.min(updatedProcessed, totalQty));

        perfAsset.setQtyProcessedNew(updatedProcessed);
        perfAsset.setRemainingStatus(updatedProcessed >= totalQty ? "Completed" : "Pending");
        performaInvoiceAssetsRepo.save(perfAsset);
    }
}
```

✅ **What this does:**
- Compares the *new* and *old* processing quantities.  
- If increased → adds the difference to `qtyProcessedNew`.  
- If decreased → subtracts the difference.  
- Prevents negative or overflow.  
- Updates remaining status (`Completed` / `Pending`).  

---

### 🔹 **Non-PO Flow — Update TRequest Asset**

Same logic but applied to request table:
```java
else if ("Non-PO".equalsIgnoreCase(poNonPo)) {
    TRequestAssetEntity reqAsset = tRequestAssetRepo.findByReqAssetId(row.getInvAstId());
    if (reqAsset != null) {
        ...
        reqAsset.setInvQtyProcessed(updatedProcessed);
        reqAsset.setInvRemainStatus(updatedProcessed >= totalQty ? "Completed" : "Pending");
        tRequestAssetRepo.save(reqAsset);
    }
}
```

---

### 4️⃣ **Issue That Triggered This Logic**

Before this fix:
- Changing the processing quantity from Invoice Validation page caused wrong updates.
- Example:
  - If user **increased** qty in Invoice Validation, performa got incremented correctly.
  - But if user **decreased**, it *didn’t reduce back* properly.
- Because backend logic didn’t consider **difference (delta)** between old and new qty.

**Now fixed** with delta calculation in service layer.

---

## ✅ **Final Outcome**

| Area | Change | Impact |
|------|---------|--------|
| **API** | `getRemainingQtyByPo/{poId}` | Fetches live parent data for validation |
| **Frontend** | Validation before updating processing qty | Prevents invalid over-entries |
| **Backend** | Adjust delta (increase/decrease) | Keeps both tables consistent |
| **UI Experience** | Toast warnings + live limits | Better user guidance |
| **Data Integrity** | Guaranteed sync between Performa & Invoice | No more mismatched quantities |

---

## 🧩 **Key Learning**

- **Parent-child data flow:** Always fetch live parent data if the child can exist independently.
- **Delta-based updates:** Compare *old* vs *new* instead of blindly replacing.
- **Multiple invoices same PO:** Need strong linkage logic (via `invAstId`).
- **Frontend + Backend sync:** Frontend stops invalid entry, backend ensures DB consistency.

---

### 🧱 **Files Changed**
| Layer | File | Key Change |
|-------|------|------------|
| Frontend | `/views/Invoice/UploadInvoicePage.tsx` | Added fetchRemainingQty(), handleEditChange() validation |
| Backend Controller | `/controller/UploadInvoiceNewController.java` | Added `upsertUploadInvoiceAssets` endpoint |
| Backend Service | `/service/UploadInvoiceNewService.java` | Added delta comparison logic + performa sync |

---

### ✅ **End Result**
System now handles multiple invoices for same PO **accurately**,  
reflects real-time remaining quantities, and maintains perfect data sync  
between **PerformaInvoiceAssets** and **UploadInvoiceAssets**.
