---
name: toplogistics-customs-upload
description: Automate customs inspection document uploads to TLA IMS (Top Logistics Australia) held shipment portal. Use when the user needs to upload PDF invoices/documents to https://ims.toplogistics.com.au for customs-held shipments, or when working with Australian customs clearance files for Cainiao/4PX logistics operations. Also use when the user asks about improvements, roadmap, future enhancements, or optimization ideas for the upload tool. Covers filename parsing strategies, captcha automation, logging/monitoring, cloud config, retry logic, and LLM integration.
name_zh: TLA海关文件上传
---

# TLA IMS Customs Document Upload Tool

## When to Use

- User needs to upload PDF files to TLA IMS for customs-held shipments
- Working with Australian customs inspection documents
- Processing held shipment lists and attaching invoices
- User mentions "海关查验", "TLA", "Top Logistics", "held shipments", "customs files"
- User asks about improvements, roadmap, or optimization of this tool

## Project Architecture

### Distribution Package

The tool is packaged as a portable Windows distribution:

```
TLA海关查验上传1.0/
├── 启动上传工具.bat          ← User entry point (double-click to run)
├── 查看运行报表.bat          ← View historical HTML report
├── 运行说明.txt              ← Usage instructions
└── _internal/               ← Hidden folder (attrib +h), contains all program files
    ├── TLA海关查验上传.exe   ← Main automation exe (PyInstaller --onefile)
    ├── 查看运行报表.exe      ← Standalone report generator exe
    ├── chromium-1067/        ← Portable Playwright Chromium browser
    └── upload_config.json    ← Saved config (pdf_dir, log_dir only, NO credentials)
```

### Key Source Files

- **auto_upload_final.py** — Main automation script, includes:
  - `get_app_dir()` — Detects exe vs script mode for portable paths
  - `JsonLogger` class — Structured JSON logging per batch
  - `generate_html_report()` — Embedded HTML report with Excel export (SheetJS)
  - `find_chromium()` — Chromium search: bundled → portable → system-installed
  - `load_or_ask_config()` — Config page with temp-path fallback for Chinese paths
  - `move_file()` — Moves PDFs to batch-timestamped subfolders
- **config.html** — Compact config UI (top-bar layout, no gradient background)
- **generate_report.py** — Standalone report generator (also built into main exe)

### Build Commands

```bash
# Main exe (bundles config.html inside)
py -m PyInstaller --onefile --add-data "config.html;." --name "TLA海关查验上传" auto_upload_final.py --noconfirm

# Report generator exe
py -m PyInstaller --onefile --name "查看运行报表" generate_report.py --noconfirm
```

### BAT Launcher Requirements

**CRITICAL**: BAT files MUST use CRLF line endings (`\r\n`). The Write tool outputs LF-only, which causes bat files to silently fail on Windows. Always use a Python script with `newline='\r\n'` to write bat files:

```python
with open(path, 'w', encoding='utf-8', newline='\r\n') as f:
    f.write('\n'.join(lines) + '\n')
```

BAT sets `PLAYWRIGHT_BROWSERS_PATH` to the `_internal` folder so the exe finds portable Chromium:

```bat
set "PROG_DIR=%~dp0_internal"
set "PLAYWRIGHT_BROWSERS_PATH=%PROG_DIR%"
"%PROG_DIR%\TLA海关查验上传.exe"
```

**CRITICAL**: Never use Chinese full-width parentheses `（）` in folder names used by bat files — cmd.exe treats them as syntax characters and crashes silently.

## Security Design

**Credentials are NEVER saved to disk.** Only `pdf_dir` and `log_dir` are persisted in `upload_config.json`. Username and password must be entered on every run through the config page.

## Core Workflow

### 0. Configuration Page

Compact HTML form (no large gradient background — previously caused display issues in small windows). Uses top-bar layout with inline username/password row.

```python
def load_or_ask_config(page, browser):
    # Build URL with saved pdf_dir and log_dir as query params
    page.goto(f"file:///{config_path}?pdf_dir={saved_pdf_dir}&log_dir={saved_log_dir}")

    # FALLBACK: If path contains Chinese characters, browser may close ("Target closed")
    # Copy config.html to tempfile.gettempdir() and retry from there
    except Exception:
        temp_html = os.path.join(tempfile.gettempdir(), "tla_config.html")
        shutil.copy2(CONFIG_HTML, temp_html)
        page.goto(f"file:///{temp_html}?...")

    page.wait_for_selector("text=信息已确认", timeout=600000)

    # Read values including log_dir
    config = page.evaluate("""() => ({
        pdf_dir, log_dir, username, password
    })""")

    # ONLY save pdf_dir and log_dir — NEVER save credentials
    json.dump({"pdf_dir": ..., "log_dir": ...}, f)
```

### 1-7. Upload Workflow

Same as before — see reference.md for full selector details. Key flow:
1. Navigate to heldList.app (direct goto, not SPA button click)
2. Login with captcha (user enters manually, prompt shown at top-left corner)
3. Search by tracking number in `#task_grid_view`
4. Click VIEW → modal opens
5. Click "Upload Document" → popup page
6. Upload via plupload file input
7. Duplicate detection via uploaded docs table

### Login Prompt

Small top-left banner instead of large centered overlay (previously blocked captcha):

```python
div.style.position = 'fixed';
div.style.top = '10px'; div.style.left = '10px';
div.style.padding = '12px 18px';
div.style.fontSize = '14px';
div.innerHTML = '【请操作】请输入验证码并点击 Login';
```

### Batch Folder Structure

After processing, PDFs are moved into a timestamp-named batch folder under the source PDF directory:

```
PDF文件夹/
├── 20260426_1605/
│   ├── 已上传/
│   │   ├── SPXMEL089400222563.pdf
│   │   └── SPXMEL089400224629.pdf
│   └── 官网查不到/
│       └── 00393580940138045945.pdf
├── 20260427_0930/
│   ├── 已上传/
│   └── 官网查不到/
└── （未处理的PDF留在根目录等待下次运行）
```

```python
# Created at Step 4 start, before processing loop
BATCH_DIR = os.path.join(PDF_DIR, time.strftime("%Y%m%d_%H%M"))
os.makedirs(BATCH_DIR, exist_ok=True)

# All move_file calls use BATCH_DIR instead of PDF_DIR
move_file(pdf_path, "已上传", BATCH_DIR)
move_file(pdf_path, "官网查不到", BATCH_DIR)
```

`get_pdf_files()` uses non-recursive `glob("*.pdf")` so it only picks up root-level PDFs, not files already sorted into batch folders.

## JSON Logging

`JsonLogger` class writes structured JSON per batch run:

```python
class JsonLogger:
    def __init__(self, log_dir, pdf_dir, username):
        self.batch_id = time.strftime("run_%Y%m%d_%H%M%S")
        self.data = {
            "batch_id": ..., "started_at": ..., "ended_at": None,
            "total_files": 0, "success_count": 0, "failure_count": 0, "skipped_count": 0,
            "pdf_dir": ..., "username": ...,
            "steps": {
                "download": {"status": "not_implemented", "files": []},
                "process": {"status": "not_implemented", "files": []},
                "upload": {"status": "in_progress", "files": []}
            }
        }

    def start_file(self, filename, tracking): ...
    def end_file(self, status, destination=None, error=None): ...
    def finish(self, overall_status="completed"): ...  # Writes run_YYYYMMDD_HHMMSS.json
```

## HTML Report with Excel Export

Auto-generated after each run. Features:
- **Top bar** with summary stats and "导出 Excel" button
- **Two tabs**: "运行概览" (card view by batch) and "明细数据" (flat table with search)
- **Excel export** via SheetJS (cdnjs CDN), columns: 单号、文件名、上传结果、上传时间、耗时(秒)、去向、备注
- File named `TLA_上传记录_YYYYMMDD_HHMM.xlsx`
- JSON data embedded as `const FILE_DATA = [...]` in the HTML for client-side export

## PyInstaller Pitfalls

1. **`exit()` is undefined** in frozen exe — always use `sys.exit(1)`
2. **`get_app_dir()`** must use `os.path.dirname(sys.executable)` when `getattr(sys, 'frozen', False)`, not `__file__`
3. **`file:///` URLs with Chinese characters** cause browser "Target closed" — use temp-path fallback
4. **`resource_path()`** uses `sys._MEIPASS` for bundled files (config.html)

## Key Selectors Reference

| Element | Selector |
|---------|----------|
| Username input | `input[name="LoginForm[user]"]` |
| Password input | `input[name="LoginForm[pwd]"]` |
| Captcha input | `input[name="LoginForm[vvc]"]` |
| Ref search box | `#task_grid_view input#ImParcel_ref` |
| VIEW button | `a.tracking-modal-link` |
| Upload link (modal) | `a:has-text('Upload Document')` |
| File input (plupload) | `input[type='file']` |
| Start Upload button | `a.plupload_start:not(.plupload_disabled)` |
| Modal body | `#modal-tracking .modal-body` |
| Uploaded docs table | `#_import_custom-grid` |

## Common Pitfalls

1. **Login wait exits early** — Use `continue` on exceptions, never `break`
2. **Ref input matches 2 elements** — Always scope with `#task_grid_view`
3. **Start Upload not found** — It's an `<a>` tag with class `plupload_start`, not `<button>`
4. **Page load timeouts** — Use `wait_until="domcontentloaded"` instead of default "load"
5. **Held Shipments button unreliable** — Direct `goto` is more reliable than clicking Dashboard buttons
6. **BAT file silent failure** — Must be CRLF; never use Chinese full-width parentheses in paths
7. **Config page not visible** — Use compact top-bar layout, not centered card on gradient

## Roadmap & Improvement Ideas

When the user asks about improvements or optimization, present this checklist.

### Phase 1: Zero-Cost (Implemented)

- [x] Structured JSON logging per batch with extensible steps
- [x] HTML report with stats dashboard and per-file details
- [x] Excel export from HTML report (SheetJS)
- [x] Batch folder organization (timestamp-named with 已上传/官网查不到 subfolders)
- [x] Portable distribution package with hidden _internal folder
- [x] Config page with compact UI and temp-path fallback
- [x] Non-blocking login prompt (top-left, doesn't block captcha)

### Phase 1b: Zero-Cost (Not Yet Implemented)

- [ ] Filename tracking number extraction (heuristic rules + configurable suffix list)
- [ ] Per-file retry logic (3 attempts before marking as failed)
- [ ] `上传失败` subfolder for files that error during upload (distinct from 官网查不到)
- [ ] Config cloud sync (read from shared network drive)

### Phase 2: Budget-Dependent

- [ ] LLM-powered filename parsing (~¥0.005/file)
- [ ] Captcha auto-recognition (ddddocr local or 2captcha API)
- [ ] Upload completion notifications (DingTalk/email)

### Phase 3: Advanced

- [ ] Full web dashboard with drag-and-drop, real-time progress
- [ ] Multi-account support
- [ ] Headless mode with scheduled runs
- [ ] Online deployment (WebSocket-based captcha relay for true web service)

### Trigger Keywords

"这个功能有什么可以改进的地方", "还有什么可以优化的", "上传工具的改进方案", "roadmap", "未来可以做什么", "优化建议"

---

## Additional Resources

- For complete page structure analysis, see [reference.md](reference.md)
- For usage examples and running instructions, see [examples.md](examples.md)
