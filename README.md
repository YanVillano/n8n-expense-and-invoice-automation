# Automated Invoice Processing & Notification System

## Overview

This project is a **complete invoice automation workflow** built using **n8n**, **Google Drive**, **Google Sheets**, **OpenAI**, and **Slack**.  

It automates:

1. **Invoice intake** – automatically detects new invoices uploaded to Google Drive.  
2. **PDF validation** – rejects non-PDF uploads.  
3. **Data extraction** – OpenAI parses invoice PDFs into structured JSON.  
4. **Record keeping** – appends invoice data into Google Sheets.  
5. **Due date monitoring** – checks invoices daily for upcoming due dates.  
6. **Notifications** – alerts the finance team in Slack for invoices due tomorrow.  

This workflow reduces manual follow-ups, prevents overdue invoices, and ensures timely payments.

---

## Workflow 1 – PDF Upload & Processing

**Flow Diagram:**

```
GDrive Trigger (New File)
      ↓
IF Node → Is PDF?
      ↓ Yes → Google Drive Download → OpenAI Node → Google Sheets Append Row
      ↓ No  → Google Drive Move File → Rejected Files Folder
```

### Details

- **GDrive Trigger:** Monitors a folder for new invoice files.  
- **IF Node:** Checks if the uploaded file is a PDF (`Mime Type = application/pdf`).  
- **Google Drive Download:** Downloads the PDF for processing.  
- **OpenAI Node:** Extracts structured JSON data from the PDF, including:  

```json
{
  "vendor": "VENDOR NAME",
  "invoiceNo": 39405,
  "date": "September/18/2026",
  "dueDate": "September/25/2027",
  "total": 10000,
  "status": "Pending"
}
```

- **Google Sheets Append Row:** Stores the invoice in the tracking sheet.  
- **Rejected Files:** Moves non-PDF uploads to a dedicated folder.

---

## Workflow 2 – Due Date Notifications

**Flow Diagram:**

```
Schedule Trigger (Every Midnight)
      ↓
Google Sheets Get Rows
      ↓
Function Node → Check if 'Due Date' is tomorrow
      ↓
Slack Node → Notify Finance Team
```

### Details

- **Schedule Trigger:** Runs daily at midnight.  
- **Google Sheets Node:** Retrieves all invoice rows.  
- **Function Node:** Calculates if an invoice is due tomorrow and skips paid invoices.  
- **Slack Node:** Sends a message to finance with invoice details.  

**Sample Slack Message:**

```
:warning: *Invoice Alert*

Vendor: *yan*
Invoice No: *54252*
Due Date: *2025-09-27*
Amount: *₱105,452*
Status: *pending*
Alert Type: *Due Tomorrow*

Please take necessary action.
```

---

## Google Sheets Structure

| Invoice Number | Vendor  | Date Created | Due Date   | Total    | Status  |
|----------------|---------|-------------|-----------|----------|---------|
| 54252          | yan     | 2025-09-25  | 2025-09-27 | 105452  | pending |
| 54253          | yanvil  | 2025-09-25  | 2025-09-27 | 52213   | pending |

---

## Function Node – Due Date Checker (JavaScript)

```javascript
const today = new Date();
today.setHours(0,0,0,0);

const result = [];

$items().forEach(item => {
    const inv = item.json;

    // Skip paid invoices
    if(inv.Status && inv.Status.toLowerCase() === "paid") return;

    // Parse Due Date
    const due = new Date(inv["Due Date"]);
    due.setHours(0,0,0,0);

    // Difference in days
    const diffDays = Math.floor((due - today) / (1000 * 60 * 60 * 24));

    let alertType = null;
    if(diffDays === 0) alertType = "Due Today";
    else if(diffDays === 1) alertType = "Due Tomorrow";

    if(alertType) result.push({ ...inv, alertType });
});

// Return items to downstream nodes
return result.map(item => ({ json: item }));
```

---

## Setup Instructions

1. **Install n8n** (Desktop, Docker, or Cloud).  
2. **Create Google Drive folders:**  
   - `Expense & Invoice Automation` → for uploaded invoice PDFs  
   - `Rejected Files` → for invalid file types  
3. **Create Google Sheets:**  
   - Columns: `Invoice Number`, `Vendor`, `Date Created`, `Due Date`, `Total`, `Status`  
4. **Configure Nodes:**  
   - GDrive Trigger → monitor folder  
   - IF Node → check PDF mime type  
   - Google Drive Download → download PDFs  
   - OpenAI Node → extract invoice JSON  
   - Google Sheets Append Row → store data  
   - Function Node → check due dates  
   - Slack Node → send notifications   
5. **Test Workflow:** Upload sample PDFs to ensure notifications trigger correctly.

---

## Future Improvements

- Aggregate multiple due invoices into one Slack message per vendor.  
- Add automatic PDF parsing without OpenAI (using OCR or PDF Parser nodes).  
- Escalation of overdue invoices to finance managers.  
- Weekly or monthly summary reports of all invoices.
