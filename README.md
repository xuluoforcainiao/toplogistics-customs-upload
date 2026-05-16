# TLA IMS Customs Document Upload Tool

Automate customs inspection document uploads to TLA IMS (Top Logistics Australia) held shipment portal, replacing the tedious manual login-search-upload workflow.

## Use Cases

- Upload PDF invoices/documents to TLA IMS for customs-held shipments
- Process Australian customs inspection documents for Cainiao/4PX logistics
- Batch process held shipment file uploads
- Get structured logs and runtime reports for traceability

## Core Features

- **Browser automation**: Auto-login TLA IMS, search tracking number, upload documents
- **Structured logging**: JSON format structured logs per run, recording processing details
- **HTML report**: Visual runtime report with batch view and Excel export
- **Batch archiving**: Auto-sort processed PDFs into timestamped folders (uploaded / not-found)
- **Zero dependency**: Embedded Chromium browser, no pre-installation required

## Security Design

- Username and password are **NEVER saved to disk**
- Only pdf_dir and log_dir persisted in config file
- Credentials entered via local HTML config page on every run

## Runtime Log Example

```json
{
  "batch_id": "run_20260426_143022",
  "total_files": 12,
  "success_count": 10,
  "failure_count": 1,
  "skipped_count": 1,
  "steps": {
    "upload": { "status": "completed", "files": [...] }
  }
}
```

## Tech Stack

- Python + Playwright (browser automation)
- PyInstaller (standalone exe packaging)
- tkinter / HTML (config UI)
- SheetJS (report Excel export)

## Related Projects

- [australia-customs-pdf-tla-suite](https://github.com/xuluoforcainiao/australia-customs-pdf-tla-suite) - Integrated suite with Excel-to-PDF tool
