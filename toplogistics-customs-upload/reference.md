# TLA IMS Page Structure Reference

## Site Architecture

### Domains
- **Main portal**: `https://ims.toplogistics.com.au` — login, dashboard, held list
- **Upload portal**: `https://os.toplogistics.com.au` — document upload (separate subdomain)

### Technology Stack
- jQuery + Bootstrap 3
- yiiGridView for data tables with AJAX filtering
- pluploadQueue for file uploads
- Single-page application (SPA) with AJAX content loading into `#main`

---

## Login Page

### Modal Structure
```html
<div class="modal fade in" id="modal-login" style="display: block;">
  <form id="login-form" action="/site/auth.app" method="POST">
    <input type="text" name="LoginForm[user]">
    <input type="password" name="LoginForm[pwd]">
    <input type="text" name="LoginForm[vvc]" autocomplete="off">
    <img id="lf_capcha" src="/site/captcha?...">
    <button type="submit" class="btn btn-primary">Login</button>
  </form>
</div>
```

### Login Behavior
- Form submits via standard POST (not AJAX)
- On success, server returns the Dashboard HTML which replaces `#main` content
- Modal remains in DOM but `display: none` after Bootstrap modal close
- Detection must handle the transition period when page is refreshing

---

## Dashboard Page

### Key Elements
- Welcome heading: `text=Welcome - 4px`
- Held Shipments tile: contains `a` linking to `/shipment/heldList.app`
- Menu dropdown: contains `a.ajax-link` entries for all modules

### Navigation Strategy
**Do NOT click the Dashboard tiles.** The SPA AJAX loading is unreliable for automation. Instead:
```python
page.goto("https://ims.toplogistics.com.au/shipment/heldList.app")
```

---

## Held Shipment List Page

### URL
`https://ims.toplogistics.com.au/shipment/heldList.app`

### Grid Structure
```html
<div id="task_grid_view" class="grid-view">
  <table class="items table table-striped table-bordered">
    <thead>
      <tr> <!-- sortable headers -->
        <th><a class="sort-link" href="...">Connote</a></th>
        <th><a class="sort-link" href="...">Ref</a></th>
        <th>...</th>
      </tr>
      <tr class="filters"> <!-- filter inputs -->
        <td><input id="ImParcel_hbn" name="ImParcel[hbn]"></td>
        <td><input id="ImParcel_ref" name="ImParcel[ref]"></td>
        <td>...</td>
      </tr>
    </thead>
    <tbody>
      <tr class="odd">
        <td>...</td>
        <td class="button-column">
          <a class="tracking-modal-link" href="/imtk/shipment/view/id/...">View</a>
        </td>
      </tr>
    </tbody>
  </table>
</div>
```

### yiiGridView AJAX Filter
- Trigger: Enter key in filter inputs
- Effect: Grid content reloads via AJAX
- Loading indicator: `.grid-view-loading` class on grid
- Global loading: `#loading-container` visible

### Duplicate Input Warning
There are **two** elements with `id="ImParcel_ref"` on the page after certain operations:
1. The grid filter input inside `#task_grid_view`
2. Another input in a different form context

Always scope: `#task_grid_view input#ImParcel_ref`

---

## View Shipment Modal

### Trigger
Clicking `a.tracking-modal-link` executes:
```javascript
$('#modal-tracking').modal();
$('#modal-tracking .modal-body').load($(this).attr('href'));
```

### Modal Structure
```html
<div class="modal fade" id="modal-tracking">
  <div class="modal-dialog modal-lg">
    <div class="modal-content">
      <div class="modal-body"></div>  <!-- loaded via AJAX -->
      <div class="modal-footer">
        <button id="modal_close" data-dismiss="modal">Close</button>
      </div>
    </div>
  </div>
</div>
```

### Modal Content
After AJAX load, `.modal-body` contains:
- Tracking/Main tabs
- Held Reason (e.g., AQIS)
- Customs Process information
- **Upload Document** link (`<a>` tag with href to upload portal)

---

## Upload Page (plupload)

### URL Pattern
```
https://os.toplogistics.com.au/parcelStatus?hbn={tracking}&ref={tracking}&h={hash}&r=aqis&awb={awb}
```

### Page Structure
```html
<div class="container">
  <div class="jumbotron">
    <h3>Parcel Number:{tracking}</h3>
    <h3><b>Reason:</b> Collect Documents for Custom AQIS declaration</h3>
    <h3><b>Status:</b> Held</h3>
  </div>

  <!-- Tabs -->
  <ul class="nav nav-tabs">
    <li class="active"><a href="#..._tab_1">Upload Documents</a></li>
    <li><a href="#..._tab_2">Goods Information</a></li>
  </ul>

  <!-- Uploaded documents grid -->
  <div id="_import_custom-grid" class="grid-view">
    <table class="items">
      <thead><tr><th>Name</th><th>Size</th><th>Date</th><th></th></tr></thead>
      <tbody>
        <tr><td><a href="...">{filename}.pdf</a></td><td>142.96 KB</td><td>2026-04-22</td><td><a class="remove_class">Delete</a></td></tr>
      </tbody>
    </table>
  </div>

  <!-- plupload component -->
  <div id="{id}_prodphoto_uploader">
    <div class="plupload_wrapper">
      <div class="plupload_filelist_header">...</div>
      <ul class="plupload_filelist">
        <li class="plupload_droptext">Drag files here.</li>
      </ul>
      <div class="plupload_filelist_footer">
        <a class="plupload_button plupload_add">Add Files</a>
        <a class="plupload_button plupload_start plupload_disabled">Start Upload</a>
        <span class="plupload_total_status">0%</span>
      </div>
    </div>
  </div>
</div>
```

### plupload Behavior
- Initialization: `$('#{id}').pluploadQueue({...})` called on page load
- Runtime: html5,flash
- Max files: 5 (enforced by `FilesAdded` event handler)
- File filter: `pdf,doc,docx,xls,xlsx,jpg,png`
- Max size: 10mb
- Upload URL: `/filerepo/upload/{hash}.app`

### File Upload Events
- `FilesAdded`: triggered when files are added (splice if > 5)
- `FileUploaded`: triggered after each file completes
  - Updates `_import_custom-grid` via `yiiGridView.update`
  - Triggers `reload_pulfile_grid` custom event
  - If `up.total.queued == 0`, reinitializes the uploader

### Hidden File Input
plupload creates a hidden moxie-shim container:
```html
<div class="moxie-shim moxie-shim-html5" style="position: absolute; ...">
  <input type="file" style="...">
</div>
```
This input is accessible via Playwright's `set_input_files`.

---

## Filename-to-Tracking-Number Extraction (Roadmap)

When PDF filenames contain non-tracking suffixes (e.g., `ME1776054438575Z2Z_merged.pdf`), the script must extract the actual tracking number before searching the grid. Three implementation strategies have been designed, selectable based on budget and accuracy requirements.

### Strategy A: Heuristic Rule Engine (Zero Cost, Offline)

**How it works**: Split filename by `_` and `-`, then score each part based on tracking-number characteristics vs. document-suffix characteristics.

**Scoring rules**:
- Contains digits → +2 (tracking numbers always contain digits)
- Length 8-20 → +1 (typical tracking number length)
- Is a known English word (merged, invoice, customs, final, copy...) → -3 (likely a suffix)
- Length < 5 → -1 (too short to be a tracking number)

**Built-in suffix dictionary**:
```python
KNOWN_SUFFIXES = {
    'merged', 'invoice', 'customs', 'final', 'copy',
    'revised', 'draft', 'new', 'old', 'original',
    'backup', 'v1', 'v2', 'v3', 'ver1', 'ver2'
}
```

**Pros**: Zero cost, fully offline, no external dependencies.
**Cons**: Accuracy depends on rule coverage; may fail on unusual suffixes or tracking numbers that contain English words.
**Estimated accuracy**: 80-95% for common naming patterns.

### Strategy B: Configurable Suffix List (Zero Cost, Offline, User-Maintained)

**How it works**: Add a text input to `config.html` where the user maintains a comma-separated list of known suffixes. The script removes these suffixes before extracting the tracking number.

**Config example**:
```
Common filename suffixes: merged,invoice,customs,final
```

**Priority**: User-configured suffix list takes precedence over heuristic rules.

**Pros**: 100% accuracy for known suffixes; user can add new suffixes without code changes.
**Cons**: Requires user to maintain the list; first time setup needed.

### Strategy C: LLM-Powered Extraction (Paid, Online, Highest Accuracy)

**How it works**: Send each filename to a cloud LLM API (OpenAI GPT-4o, Alibaba Tongyi, DeepSeek) and ask it to return only the tracking number.

**Example prompt**:
```
Extract the tracking number from this PDF filename.
Format is usually "tracking_suffix.pdf" but suffixes vary.
Return ONLY the tracking number, no explanation.

Filename: ME1776054438575Z2Z_merged.pdf
```

**Requirements**:
- Internet connection
- LLM API key (managed via config.html, not saved to disk)
- Budget for API calls (~¥0.005 per file for GPT-4o-mini)

**Pros**: 99%+ accuracy; handles any naming convention without rule maintenance.
**Cons**: Requires network; ongoing cost; API key management; latency (+0.5-3s per file); tool fails if LLM service is down.

**Hybrid fallback**: When LLM call fails (network timeout, rate limit), automatically fallback to Strategy A+B.

### Recommended Deployment Order

1. **Phase 1 (Now)**: Implement Strategy A + B hybrid. Built-in suffix dictionary covers most cases; user-configurable list handles edge cases.
2. **Phase 2 (Future, if budget approved)**: Add Strategy C as an optional enhancement. Enable via config toggle: "Use LLM for filename parsing (requires API key)".
3. **Phase 3 (Future)**: Collect misclassified filenames during Phase 1, use them to train/improve the heuristic rules or fine-tune a local model.

### How to Trigger These Strategies

When you want to implement or switch between these strategies, mention any of these keywords:
- "文件名提取" / "尾程单号识别"
- "LLM识别单号" / "大模型提取"
- "文件名后缀列表"
- "改进上传工具"
- "tracking number extraction"

This will load the `TLA海关文件上传` or `学习如何自动化操作网站` skill and retrieve the complete implementation details.

---

## Complete Selector Reference

| Purpose | Selector | Notes |
|---------|----------|-------|
| Login username | `input[name="LoginForm[user]"]` | Inside `#modal-login` |
| Login password | `input[name="LoginForm[pwd]"]` | Inside `#modal-login` |
| Login captcha | `input[name="LoginForm[vvc]"]` | Inside `#modal-login` |
| Captcha image | `img#lf_capcha` | Click to reload |
| Login button | `button[type="submit"]` | Inside `#login-form` |
| Welcome indicator | `text=Welcome` | Confirms login success |
| Ref filter input | `#task_grid_view input#ImParcel_ref` | **Scoped required** |
| Connote filter | `#task_grid_view input#ImParcel_hbn` | |
| Grid rows | `#task_grid_view table tbody tr` | |
| VIEW button | `a.tracking-modal-link` | Text is "View" |
| Modal | `#modal-tracking` | Content loaded via AJAX |
| Modal body | `#modal-tracking .modal-body` | |
| Modal close | `#modal_close` | Or press Escape |
| Upload link | `a:has-text('Upload Document')` | **Singular** |
| Popup page | `expect_popup()` | Opens new tab/window |
| File input | `input[type='file']` | plupload shim |
| Add Files | `a.plupload_add` | |
| Start Upload | `a.plupload_start:not(.plupload_disabled)` | `<a>` not `<button>` |
| Upload status | `.plupload_total_status` | "0%" → "100%" |
| Uploaded grid | `#_import_custom-grid` | yiiGridView table |
| Uploaded rows | `#_import_custom-grid table tbody tr` | For duplicate check |
| Delete link | `a.remove_class` | In uploaded grid |
