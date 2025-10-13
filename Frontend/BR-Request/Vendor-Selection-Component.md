# 🧾 Vendor File Upload & Download Flow — Internal Documentation

### 📁 Component
`VendorSelectionPro.tsx`

### 🧠 Purpose
Handles vendor selection, creation, and quotation file upload for each vendor row. Integrates with backend folder `BRRequest` for file storage and retrieval.

---

## ⚙️ Flow Summary

| Step | Action | Function | Key Field |
|------|--------|-----------|-----------|
| 1️⃣ | User uploads file from table | `handleVendorFileUpload(rowData, e.files)` | Updates `quotationUpload` |
| 2️⃣ | File sent to backend | `SingleFileUploadApi([file], 'BRRequest')` | Returns uploaded filename |
| 3️⃣ | Filename stored in UI state | `setNewVendors()` and `setAddedVendors()` | `quotationUpload` |
| 4️⃣ | UI displays FileUpload component | Inside `MasterDataTable → anchorColumn` | Bound to `rowData.quotationUpload` |
| 5️⃣ | User clicks download | `anchorActionSupport(fileName)` | Calls `FileDownload(fileName, 'BRRequest')` |
| 6️⃣ | Backend serves file | `/api/v1/smbfiledownload/BRRequest/{fileName}` | Returns attachment |

---

## 🧩 Core Functions

### 1. File Upload
```ts
const handleVendorFileUpload = async (rowData: any | null, files: any) => {
  if (!files || files.length === 0) return;
  try {
    const file = files[0];
    const uploadedFileName = await SingleFileUploadApi([file], 'BRRequest');

    if (rowData) {
      // Update quotationUpload field in both newVendors and addedVendors
      setNewVendors(prev => prev.map(v =>
        v.id === rowData.id ? { ...v, quotationUpload: uploadedFileName } : v
      ));
      setAddedVendors(prev => prev.map(v =>
        v.id === rowData.id ? { ...v, quotationUpload: uploadedFileName } : v
      ));
    } else {
      // File uploaded in Add/Edit vendor dialog
      setUploadedFileName(uploadedFileName);
    }

    toast.current.display('success', 'File uploaded successfully');
  } catch (error) {
    console.error('Error uploading vendor document:', error);
    toast.current.display('error', 'Failed to upload file');
  }
};
```

### 2. File Download
```ts
const anchorActionSupport = (fileName: string) => {
  if (!fileName || fileName === '-') {
    console.log('No file available for download');
    return;
  }
  console.log('Downloading file:', fileName);
  FileDownload(fileName, 'BRRequest');
};
```

---

## 🧠 Common Pitfalls & Fixes

| Issue | Root Cause | Fix |
|-------|-------------|-----|
| File not visible in UI | Used `documentFileName` instead of `quotationUpload` | Always use `rowData.quotationUpload` |
| File uploads but not saved | Didn’t update `newVendors` and `addedVendors` | Ensure both arrays are updated |
| File doesn’t download | Passed object instead of string to download function | `anchorActionSupport(fileName: string)` |
| Backend 404 on download | Folder name mismatch | Ensure folder = `'BRRequest'` |

---

## 🧱 UI Integration Example

```tsx
<MasterDataTable
  data={newVendors}
  Columns={vendorColums}
  Edit={EditRow}
  removeFilter
  Delete={deleteNewDataRow}
  anchorColumn={{
    field: 'quotationUpload',
    header: 'Quotation Upload',
    body: (rowData: any) => (
      <div className={styles.unclippedFileUploadCell}>
        <FileUpload
          label=""
          fileTypes={['pdf', 'jpg', 'jpeg', 'png']}
          onChange={(e) => handleVendorFileUpload(rowData, e.files)}
          downloadFileName={rowData.quotationUpload}
          downloadFunction={() => anchorActionSupport(rowData.quotationUpload)}
          mode="basic"
        />
      </div>
    )
  }}
/>
```

---

## 🧰 Related APIs

| Purpose | Method | Endpoint | Example |
|----------|---------|-----------|----------|
| Upload | POST | `/api/v1/smbfileupload/BRRequest` | From `SingleFileUploadApi` |
| Download | GET | `/api/v1/smbfiledownload/BRRequest/{fileName}` | Used in `FileDownload()` |

---

## 🧾 Best Practices

- Always use `quotationUpload` as the single source of truth.
- Match upload/download folder names (`BRRequest`).
- For new modules, rename both upload + download folder consistently.
- Avoid inconsistent naming (`documentFileName`, `quotationDoc`, etc.).
- Validate file presence before attempting download.
- Keep reusable helper for file handling across modules.

---

**Author:** Internal Documentation by Shivapunithan & ChatGPT  
**Last Updated:** October 2025  
