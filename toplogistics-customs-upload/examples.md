# Usage Examples

## Running the Upload Script

### Step 1: Prepare Files
Place PDF files in the source folder:
```
C:\Users\浩正\Desktop\澳大利亚提供资料\4PX海关查验订单发票\
├── 6013026525458.pdf
├── 6021126282399.pdf
└── ...
```

### Step 2: Run the Script
Open Command Prompt and execute:
```cmd
py "C:\Users\浩正\.qoderwork\workspace\mo8f7bwhwjtu5lih\auto_upload_final.py"
```

### Step 3: Configure (First Run)
A browser window opens a local configuration page:
- **PDF 文件夹路径**: Enter the folder containing PDF files (saved for next time)
- **TLA 用户名**: Enter username (**not saved**, must re-enter every run)
- **TLA 密码**: Enter password (**not saved**, must re-enter every run)
- Click **开始上传**

On subsequent runs, the PDF path is pre-filled; only username and password need to be entered.

### Step 4: Enter Captcha
After configuration, the TLA login page loads. The script auto-fills the username and password you just entered. **Manually enter the captcha and click Login.**

### Step 4: Wait for Completion
The script will automatically:
1. Navigate to Held Shipment List
2. Search each PDF filename as a tracking number
3. Click View → Upload Document
4. Upload the file (with duplicate detection)
5. Move processed files to subfolders

---

## Example: Successful Upload Flow

```
[19:48:25] Opening heldList.app...
[19:48:32] Filling credentials...
[19:48:32] 请在浏览器里输入验证码并点击 Login
[19:48:32] 600秒超时...
[19:48:45] Login successful!
[19:48:48] Screenshot: login_done.png
[19:48:48] Navigating to Held Shipment List...
[19:49:02] Table loaded!

==================================================
Processing: 6013026525458
==================================================
[19:49:06] Searching...
[19:49:10] Result rows: 1
[19:49:10] Clicking View...
[19:49:15] Looking for Upload Documents Link...
[19:49:15] Found upload link with: a:has-text('Upload Document')
[19:49:15] Clicking upload link...
[19:49:15] Popup opened: https://os.toplogistics.com.au/parcelStatus?hbn=...
[19:49:20] Looking for file upload input...
[19:49:20] Found 1 file input(s)
[19:49:20] File selected.
[19:49:23] Clicking Start Upload...
[19:49:23] Waiting for upload to complete (30s)...
[19:49:53] Upload step finished.
[19:49:54] Moved to [已上传]: 6013026525458.pdf
```

---

## Example: Duplicate Detection

When a file already exists in the upload portal:

```
Processing: 6021126282399
==================================================
[19:50:05] Found upload link with: a:has-text('Upload Document')
[19:50:10] Checking for existing uploaded files...
[19:50:10] WARNING: 6021126282399.pdf already uploaded! Skipping to avoid duplicate.
[19:50:11] Upload page closed.
```

The file is **NOT** uploaded again, and the script moves to the next PDF.

---

## Example: No Results Found

When a tracking number is not in the held shipment list:

```
Processing: UNKNOWN1234567
==================================================
[19:51:00] Searching...
[19:51:05] Result rows: 0
[19:51:05] No results found.
[19:51:06] Moved to [官网查不到]: UNKNOWN1234567.pdf
```

The file is moved to the `官网查不到` folder for manual review.

---

## File Organization After Processing

```
4PX海关查验订单发票/
├── 官网查不到/          # Tracking numbers not found on portal
│   └── SPXMEL089400332997.pdf
├── 已上传/              # Successfully uploaded files
│   ├── 6013026525458.pdf
│   └── 6021126282399.pdf
└── (remaining PDFs)     # Not yet processed
```

---

## Troubleshooting

### Browser window doesn't appear
Run the script directly in Command Prompt (not via background process):
```cmd
py "C:\path\to\auto_upload_final.py"
```

### Login times out in 24 seconds
This was a bug in earlier versions where `except: break` killed the wait loop. Ensure you're using the latest script with `except: continue`.

### "Strict mode violation: locator resolved to 2 elements"
The `input#ImParcel_ref` selector matched multiple elements. The fix is to scope it: `#task_grid_view input#ImParcel_ref`.

### Start Upload button not found
Older versions looked for `<button>`. The correct element is `<a class="plupload_start">`.

### Upload shows 0% with red error icon
This is normal before clicking Start Upload. Once Start Upload is clicked, the file uploads and reaches 100% with a green checkmark.
