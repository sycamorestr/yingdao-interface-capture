---
name: yingdao-interface-capture
description: Capture logged-in business back-office interfaces and output runnable Yingdao webpage JavaScript. Use when Codex must inspect a live backend page, prefer stable API/runtime data extraction, verify request parameters and response fields, handle cookies/signatures/webpack/mtop, or review/fix Yingdao JS that fetches business data. Only write page-operation JS when the user explicitly asks to click, type, select, switch tabs, choose dates, or otherwise manipulate UI controls.
---

# Yingdao Interface Capture

## Prime Directive

Prefer interface capture over UI automation. The most stable Yingdao script is usually a page-executed script that calls the same API, request runtime, webpack module, mtop client, or already-loaded state that the backend page uses.

Only write JavaScript that operates page controls when the user explicitly asks for page operations such as clicking, filling, selecting, switching tabs, opening date pickers, downloading through UI, or verifying visible control state. Do not fall back to simulated clicks merely because an interface is inconvenient to find.

## Operating Standard

- Use live evidence, not guesses.
- Use `opencli` for live page binding, state, network/runtime inspection, and page-context evaluation.
- Treat `opencli` browser access as a single-session lifecycle. Bind the existing user tab before every page command, reuse one session name for the entire task, and always unbind it when finished.
- Never open `about:blank` as part of exploration.
- Never call `opencli browser <session> open ...` unless the user explicitly asks to open a URL.
- Do not blindly bind by session name. Bind only after the user has made the target tab active, or after selecting an existing tab by URL/title through a tab-listing mechanism.
- After binding, immediately confirm URL and title. If they do not match the user-specified page, stop and report the mismatch before running state, frames, network, or eval commands.
- Start by binding the user-specified active page and confirming URL, title, visible state, and iframe situation.
- Prefer page-context JavaScript over standalone HTTP when cookies, CSRF tokens, dynamic signatures, anti-replay parameters, webpack request clients, or mtop runtimes are involved.
- Install request listeners before triggering any read-only UI query when the endpoint is unknown.
- Keep exploration read-only unless the user asked for a write action.
- Return a runnable Yingdao webpage-executed JS function as the final artifact, not curl notes or a standalone HTTP configuration.

## Browser Binding Safety

When the user has already opened the target business page, treat that tab as user-owned state. Do not create a new tab, navigate a tab, or use a fresh blank tab to probe the site.

Before any browser inspection:

- Ask the user to bring the exact target tab to the foreground, unless a reliable existing-tab selector is available.
- If a tab-listing command is available, list existing tabs and select the one whose URL/title matches the user's page. Do not create a tab to list or select tabs.
- Bind the session only to that foreground or selected existing tab.
- Immediately run a minimal URL/title check.
- Continue only when the URL/title matches the requested page. If the bound page is `about:blank` or any unrelated page, unbind if appropriate and stop.

## OpenCLI Session Hygiene

Prevent Edge from accumulating saved `OpenCLI Browser` tab groups. An unbound browser session creates an OpenCLI-owned window and orange tab group when its first page command runs. Closing that window or its last tab can leave a closed saved group in Edge. A successfully bound existing user tab is a borrowed session and does not need that owned group.

Apply these rules as hard requirements:

- Use one stable session name for the whole task, for example `yingdao_capture`. Do not generate a new session name for retries, phases, endpoints, or subpages.
- Make `opencli browser <session> bind` the first browser-session action. Do not run `state`, `frames`, `network`, `eval`, `get`, `tab`, or any other page command until `bind` has succeeded.
- Treat `edge://`, `chrome://`, browser new-tab pages, and extension pages as non-debuggable. If `bind` reports `bound_tab_not_found`, ask the user to foreground the actual HTTP(S) business page and retry `bind`; never continue with an unbound command.
- If `bind` reports an existing or stale binding, run `unbind` and then `bind` again with the same session name. Do not work around a binding error by choosing a new name.
- Prefer asking the user to foreground the existing target tab. Never probe with `open`, `about:blank`, `tab create`, or an unbound page command.
- Reuse the same successful binding across all inspection steps. Do not repeatedly bind/unbind between `state`, `frames`, `network`, and `eval` calls.
- Use bounded network capture whenever possible. Do not leave `opencli ... network --follow` running in the background. If follow mode is necessary, retain its process/cell handle, stop it explicitly, and only then unbind.
- Put `opencli browser <session> unbind` in the orchestration finalizer so it runs after success, command failure, timeout, or user interruption. Use `unbind`, not `close`, for a user-owned tab; do not close the user's tab or Edge window.
- After unbinding, do not issue another page command with that session unless the task explicitly resumes and the target tab is bound again first.
- If new groups still appear despite a verified bound-session workflow, inspect Edge for multiple enabled OpenCLI extension installations and report the duplicate installation. Do not silently create additional sessions.

Required lifecycle:

```text
foreground existing target tab
-> bind one stable session
-> verify URL/title
-> inspect with the same session
-> stop any follow/listener process
-> unbind in finalizer
```

## Workflow

1. Bind and confirm:

```bash
# User first activates the exact target tab, or an existing tab is selected by URL/title.
opencli browser yingdao_capture bind
opencli browser yingdao_capture eval "({href:location.href,title:document.title})"
```

Only after URL/title match the requested page:

```bash
opencli browser yingdao_capture state
opencli browser yingdao_capture frames
```

Run every later command against `yingdao_capture`, then finish with:

```bash
opencli browser yingdao_capture unbind
```

2. Identify the data path in this order:

- Already-loaded page state or React/Vue props.
- Page business functions, SDK clients, request wrappers, webpack modules, mtop runtimes, or app-specific API modules.
- Captured `fetch` / `XMLHttpRequest` requests and responses.
- Same-origin `fetch` from page context with `credentials: "include"`.
- UI automation only when explicitly requested.

3. Verify with a small real query:

- Confirm exact request fields, date formats, pagination, batch limits, and response paths.
- Confirm units such as cents versus yuan, signed versus absolute amounts, and percent formatting.
- Confirm missing-result behavior is represented as data, not a script failure, when appropriate.

4. Deliver the final Yingdao script:

- Shape must be `async function (element, input) { ... }` or `function (element, input) { ... }` as required by the task.
- Normalize `input`; Yingdao often passes quoted scalar strings such as `"2026-07-01"`.
- Return `JSON.stringify(...)`.
- Include diagnostics: `ok`, `blocked`, `message` or `reason`, `rawInput`, parsed inputs, `href`, and relevant module/API/log fields.
- Put machine-readable results in `data` when returning multiple records.

## Interface Capture Patterns

Install temporary request capture before a user or script triggers the smallest read-only query:

```js
(function () {
  window.__apiLogs = [];
  var oldFetch = window.fetch;
  window.fetch = function (input, init) {
    var url = typeof input === "string" ? input : input && input.url;
    var rec = { kind: "fetch", url: String(url || ""), method: String(init && init.method || "GET"), requestBody: init && init.body ? String(init.body) : "" };
    window.__apiLogs.push(rec);
    return oldFetch.apply(this, arguments).then(function (resp) {
      rec.status = resp.status;
      rec.contentType = resp.headers && resp.headers.get ? resp.headers.get("content-type") || "" : "";
      try { resp.clone().text().then(function (t) { rec.responseText = t.slice(0, 5000); }); } catch (e) {}
      return resp;
    });
  };
  var OldXHR = window.XMLHttpRequest;
  window.XMLHttpRequest = function () {
    var xhr = new OldXHR();
    var rec = { kind: "xhr", method: "", url: "", requestBody: "" };
    var open = xhr.open;
    xhr.open = function (method, url) { rec.method = String(method || ""); rec.url = String(url || ""); return open.apply(xhr, arguments); };
    var send = xhr.send;
    xhr.send = function (body) {
      rec.requestBody = body ? String(body) : "";
      window.__apiLogs.push(rec);
      xhr.addEventListener("loadend", function () {
        rec.status = xhr.status;
        try { rec.responseText = String(xhr.responseText || "").slice(0, 5000); } catch (e) {}
      });
      return send.apply(xhr, arguments);
    };
    return xhr;
  };
  return { ok: true };
})()
```

When webpack module ids are unstable, expose `__webpack_require__` and scan module source for endpoint names, service names, or request paths. Do not hard-code a module id when live evidence shows ids change across builds.

When mtop or signed platform APIs appear, do not replay a fixed signed URL. Prefer the page's loaded runtime and compare exact parameter types, `dataType`, `ttid`, and wrapper placement.

If direct page `fetch` returns login, risk, illegal request, empty token, or `403 {}`, use the page runtime/module or already-loaded state instead of repeated retries.

## Douyin Shop Coupon Lessons

For Douyin Shop / 抖店 coupon management work, prefer interface capture and page-context `fetch` over UI clicking. The coupon management page and the coupon copy/create page can expose different parts of the workflow, so inspect both when needed, but do not assume the final automation must operate both pages.

Useful lessons from live coupon replacement work:

- The management page can query, void, and create coupons through same-origin interfaces using the logged-in page context. After capturing the create payload from the copy/create page, the final create request can often be replayed from the management page without switching to the newly opened tab.
- Do not assume destructive verbs from REST-looking URLs. A coupon void action that looked like it belonged to `/marketing/coupons/v1/shopcoupons/{coupon_meta_id}` was not `DELETE`; the live UI used `POST` with body `{"action":1}`. Always capture the real request from the confirmation modal before coding the write action.
- The copy/create page is still valuable as a parameter oracle. Capture the exact `/marketing/coupons/v1/check` and `/marketing/coupons/v1/shopcoupons` payloads after setting the desired form values, then generalize only the fields proven to vary, such as coupon name/product id, discount, time range, goods list, and renewal settings.
- Low-price/risk prompts may appear in the UI after `/marketing/coupons/v1/check`, but the final create endpoint may still accept the same payload directly. Verify with one user-approved real submit before relying on direct interface creation. If a stronger risk control, CAPTCHA, or secondary verification appears, stop and report the blocker.
- The list filter matters: `status=3` returns active coupons, while an "all" query can find voided coupons that are still useful as copy templates. If no active coupon exists, query all statuses before returning "no data".
- Multiple active coupons with the same coupon name/product id require a deliberate policy. For replacement workflows, void all active coupons only after the create precheck succeeds, then create one new coupon. Always run a final verification query afterward because partial success can leave "voided but not recreated" rows.
- Skip replacement when active coupon discounts already equal the target discount. This avoids unnecessary void/create churn and reduces risk-control exposure.
- For product coupons, force the goods scope and product binding explicitly when the business rule requires it. In the observed payload this meant `goodsScope: 1`, `support_type: 1`, and `goods: [productId]`.
- Validate time semantics from the live payload, not labels alone. In the observed Douyin coupon copy page, "使用时间与领取时间相同" corresponded to `period_type: 1`. When the template was not effective today, the replacement workflow set both apply and use ranges from now to roughly three months later.
- Input normalization matters in Yingdao. Python expressions may pass a JSON string that arrives in page JS as a quoted JSON string, or even as a non-standard outer-quoted string. Final scripts should parse at least standard JSON, double-encoded JSON, and outer-quoted JSON fallbacks.
- For batch writes, prefer a dry-run/precheck mode, an execute mode, and a separate verification script. The verification script should return a stable table-shaped result, for example `[[productId, targetDiscount, status]]`, preserving input order for spreadsheet write-back.

## Backend Export Tasks

For business exports, separate task creation from file download. Many back-office pages create an asynchronous export task first, then expose the finished file through an export-log endpoint or page. Do not assume a browser download dialog means success; Yingdao scripts should return a machine-readable download URL or task id when possible.

- Prefer the page's own export runtime (`_ACP`, request wrapper, SDK method, webpack module, WebForms callback, etc.) over clicking export buttons.
- Capture the raw XHR/fetch response around the export call. Some runtimes pass `{}` or a normalized placeholder to the callback while the real response text contains `ClientScript`, `ReturnValue`, `autoId`, or a task id.
- Treat `ReturnValue: true` / success-without-id as "task likely created", not as failure. Fall back to the export log and look for a new task created after the script start time.
- Before creating the task, record a baseline such as max task id, existing log row count, and timestamp. When polling the log, match by task id greater than baseline, export type/name, status not failed/deleted/cancelled, and creation time near the script start.
- If multiple users can export under the same account, report that export-log fallback can misattribute concurrent tasks unless the API returns a task id or the request carries a unique filename/remark.
- Poll until the file is generated; task creation and file availability are different states. Return a clear diagnostic when the task exists but the file URL is not ready within the timeout.
- Keep `downloadUrl`, `savePath`, `taskId`/`autoId`, row count, filter summary, and `fromExportLog` in the final JSON.

When page state must be pushed into an export page, prefer the application's own state-transfer functions over hidden iframes or manual DOM cloning. For example, if the list page opens an export tab and then pushes current filters, call that list-page function; loading the export URL directly may create a valid page with empty/default filters and accidentally export the wrong dataset.

## Jushuitan Sales Subject Finance Export Lessons

These rules come from a live 聚水潭 “销售主题分析（财务）” export of the **明细（订单商品）** report. Use them only after confirming that the logged-in page exposes the same report route and runtime functions; report configurations may vary by tenant and release.

### Frame topology and stable targeting

The ERP shell page (`https://www.erp321.com/epaas`) hosts the BI report in an outer iframe. The finance report page in turn hosts the filter page in `#s_filter_frame`, and after querying it creates a detail iframe whose title is `明细(订单商品)`.

- Do not hard-code generated IDs such as `iframe-349`. Select the outer report frame by its source path instead:

```xpath
//iframe[contains(@src,'/app/daas/report/subject/adsfinance/') or contains(@src,'/app/daas/report/subject/finance/')]
```

- In a Yingdao field set to **fx/expression mode**, an XPath must be a Python string expression, not bare XPath. Enter:

```python
"//iframe[contains(@src,'/app/daas/report/subject/adsfinance/') or contains(@src,'/app/daas/report/subject/finance/')]"
```

  A bare `//iframe[...]` is parsed as Python and raises a `^, syntax error`.
- Use two separately saved iframe objects: `finance_report_frame` for the outer BI report and `finance_filter_frame` for `//iframe[@id='s_filter_frame']`. Run filter setup only in `finance_filter_frame`; run export creation and polling in `finance_report_frame`.
- A page-local job created in the filter frame must be stored on `window.parent` (the outer report iframe), for example `window.parent.__jstFinanceExportJobs`. The polling script, executed in `finance_report_frame`, can then read the same job. Do not reload, navigate, close, or switch away from the outer report page while a job is running.
- After search, identify the generated frame by title/content and verify it is the `明细(订单商品)` page before exporting. Do not accidentally export a summary, a stale tab, or the filter page.

### Date input and filter semantics

The report accepts a date range as start-inclusive and end-exclusive. A one-day report is therefore `startDate = "2026-07-12"`, `endDate = "2026-07-13"`. If a user passes the same calendar date for both values, normalize it to `endExclusive = startDate + 1 day` before applying filters. Do not write a same-day exclusive end, because it produces an empty interval.

Yingdao may pass a scalar string as a quoted string, rather than JSON. A result such as `rawInput: "\"2026-07-12\""` is not a valid two-date payload. For a script designed to accept two ordinary Yingdao variables, pass this expression to the JS action:

```python
str(startDate) + "," + str(endDate)
```

Then normalize the scalar in the page script and split it once by comma. If instead the script explicitly expects an object, pass a genuine JSON object string:

```python
__import__('json').dumps({"startDate": startDate, "endDate": endDate}, ensure_ascii=False)
```

Do not use `download: False` or other unrelated fields merely to make a JSON object; keep the action input contract minimal and document it.

The observed finance report uses page functions `SetSearchDateType`, `SetSearchAfterDateType`, `SearchFilter`, and `BtnFullSearch`. Prefer those functions to raw checkbox clicks and input assignment. For the configured “明细（订单商品）” task, build conditions with the exact keys/operators the report itself uses:

```text
cost_type = 1                                      # 发货时间成本价
A.status @= WAITPAY,WAITCONFIRM,WAITFCONFIRM,WAITDELIVER,DELIVERING,SENT,QUESTION,WAITOUTERSENT,CANCELLED
E.aftertype @= 普通退货,仅退款,拒收退货,换货,补发,投诉,其它,现场退货
C.afterstatus @= CONFIRMED
A.shop_id @= <SHOP_ID_1>,<SHOP_ID_2>,...             # Replace with shop IDs confirmed from the live tenant
combinesku_type @= 2                               # 组合商品：全部
combinesku @= 1                                    # 组合商品拆分
C.item_pay_date >= startDate; C.item_pay_date < endExclusive
C.confirm_date >= startDate;  C.confirm_date < endExclusive
```

The UI can also emit `export_date_begin >= startDate` and `export_date_end < endExclusive` as companion internal conditions. Preserve them only when live inspection proves the report adds them itself; never invent field names.

### Default-condition hygiene and whitelist validation

“Only these filters” means clearing existing defaults before setting the required controls, not merely adding required filters onto the current filter state. In the observed page, the default exclusion produced this unexpected condition:

```json
{"k":"nolabels","v":"特殊单,统计排除标","c":"=","t":""}
```

Clear relevant values across the **filter document** (not just a duplicate or hidden form): checked checkboxes/radios, text/search/number/textarea inputs, selectable controls, and stale filter state such as `nolabels`, `labels`, `labels_search`, and `exceptlab_search`. Do not clear framework identity fields such as hidden IDs prefixed `__`. Then call the report's own filter builder and compare its produced conditions with a canonical expected list before calling `BtnFullSearch`.

Validation must report at least `expectedConditions`, `actualConditions`, `unexpectedConditions`, and `missingConditions`. If an unexpected condition remains, return `blocked: true` and do not query or export. This turns a hidden default into a diagnosable configuration issue instead of silently exporting the wrong population.

### Native asynchronous export runtime

On the observed detail report page, use the page's callbacks instead of a UI export click:

- Create the task through `_ACP("CreateAsyncExport", callback, payload)`.
- Poll the export log through `_ACUHW("/app/common/export/export_log.aspx", "GetExportData", callback, taskId)`.
- Inspect real callback arguments and response text. A success response may return a task id in `rv`; it can also report success without an id.

Expose progress with stable, machine-readable steps. Poll every three seconds from Yingdao; the page-local worker itself should also use short bounded waits. Recommended states are:

| State | Step | Meaning / Yingdao action |
| --- | --- | --- |
| `running` | `WAITING_DETAIL` | Wait for the queried `明细(订单商品)` iframe to exist and validate it. |
| `running` | `CREATING_EXPORT_TASK` | Calling the native asynchronous export method. |
| `running` | `WAITING_DOWNLOAD_URL` | Task exists; poll the export log. |
| `finished` | `DOWNLOAD_URL_READY` | Read `downloadUrl`; hand it to the local download step. |
| `failed` | `FAILED` | Read `message`; do not blindly retry without diagnosing it. |
| `finished` | `EXPORT_TASK_CREATED_NO_ID` | The site accepted creation but did not return an ID. Check export center/browser downloads; do not fabricate a URL. |

If `_ACP` returns only success/`ReturnValue: true` but no task id, describe the result as “likely created” rather than a false failure. A log fallback can identify a newly created task by baseline ID/time/export name, but it can be ambiguous when several people use the same account. Prefer the returned task id whenever available.

### Yingdao variable parsing and local download

The JavaScript action returns a JSON **string**. Keep the raw output (for example, `report_js_result`) and parse individual values in “设置变量” with single expressions:

```python
jobId = __import__('json').loads(start_js_result)["jobId"]
reportState = __import__('json').loads(report_js_result).get("state", "")
reportStep = __import__('json').loads(report_js_result).get("step", "")
downloadUrl = __import__('json').loads(report_js_result).get("downloadUrl", "")
reportMessage = __import__('json').loads(report_js_result).get("message", "")
```

Loop only while `reportState == "running"`; wait three seconds, then run the status script again. Never start the filter script again inside the loop, because that resets the report and page-local job.

聚水潭 can return an already absolute, signed OSS `downloadUrl`. Preserve it exactly; do not prepend the report origin or otherwise normalize an absolute URL. Signed URLs expire, so download immediately after `DOWNLOAD_URL_READY`.

Webpage JavaScript cannot reliably choose an arbitrary local file location. If the workflow needs a custom folder, use a separate Yingdao “运行程序” action after URL extraction: pass `powershell.exe` an `-EncodedCommand` generated from `downloadUrl` and `localPath`. Build the command in a preceding “设置变量” step so the URL, ampersands, Chinese path, and quotes are not damaged by command-line parsing. The encoded PowerShell should create the parent directory, call `Invoke-WebRequest -Uri $url -OutFile $path -UseBasicParsing`, verify a non-empty file, and surface the caught error before returning exit code 1. Do not depend on a local `.ps1` file when the user wants all configuration contained in the Yingdao workflow.

## Long Yingdao JS Tasks

When Yingdao's webpage "execute JavaScript" action must run inside the logged-in page context but has a fixed timeout that cannot be increased, split long work into two short scripts:

1. A start script that creates a page-local job object on `window`, starts an un-awaited async function, and returns immediately with a `jobId`.
2. A status script that reads the page-local job object by `jobId` and returns progress, partial results, and final state.

This works because the browser page continues to run its own event loop after the automation action receives the immediate return value, as long as the page is not refreshed, closed, navigated away, or frozen. The long work should use page-context `fetch`, cookies, request wrappers, and `setTimeout` delays as usual, while progress is written to a stable object such as `window.__couponBatchJobs[jobId]`.

Use this pattern only when it is acceptable for the task to live in the current page's memory. It is not durable: page refresh, navigation, browser close, or automation cleanup can destroy the job. Always provide a polling/status script, clear states such as `running`, `finished`, and `failed`, and a final verification script for business-critical writes.

Minimal start pattern:

```js
async function (element, input) {
  var jobId = "job_" + Date.now();
  window.__ydJobs = window.__ydJobs || {};
  window.__ydJobs[jobId] = { state: "running", results: [], startedAt: Date.now() };

  (async function () {
    try {
      for (var i = 0; i < rows.length; i++) {
        window.__ydJobs[jobId].currentIndex = i + 1;
        var result = await doOne(rows[i]);
        window.__ydJobs[jobId].results.push(result);
        await new Promise(function (resolve) { setTimeout(resolve, 2000); });
      }
      window.__ydJobs[jobId].state = "finished";
      window.__ydJobs[jobId].finishedAt = Date.now();
    } catch (e) {
      window.__ydJobs[jobId].state = "failed";
      window.__ydJobs[jobId].message = String(e && e.message || e);
      window.__ydJobs[jobId].finishedAt = Date.now();
    }
  })();

  return JSON.stringify({ ok: true, jobId: jobId, state: "running" });
}
```

Minimal status pattern:

```js
async function (element, input) {
  var args = typeof input === "string" ? JSON.parse(input || "{}") : (input || {});
  var job = window.__ydJobs && window.__ydJobs[args.jobId];
  if (!job) return JSON.stringify({ ok: false, message: "job not found" });
  return JSON.stringify({
    ok: job.state !== "failed",
    jobId: args.jobId,
    state: job.state,
    currentIndex: job.currentIndex || 0,
    results: job.results || [],
    message: job.message || ""
  });
}
```

## Page Operation Boundary

Use page-control logic only when the user explicitly requests it. Examples:

- "select the date and click query"
- "write Yingdao JS to operate this dropdown"
- "switch to daily summary and export"
- "this date picker cannot be filled; fix the page-operation JS"

When page operations are requested:

- Inspect frames and duplicate forms before selecting controls.
- Prefer page-supported business functions or widget APIs over raw DOM assignment.
- Treat React/Vue controlled inputs, readonly inputs, custom selects, date pickers, and portal overlays as component state, not simple `.value` strings.
- Verify before/after visible values and backing state before clicking search/export.
- Keep diagnostics in the return JSON until the user confirms the script works.

Do not include page-operation code in an interface-only extraction script unless it is needed to load the data and the user approved that behavior.

## Yingdao Compatibility

Treat the Yingdao webpage JavaScript action input as a **string-only boundary**. This is a fixed requirement for scripts and usage instructions produced by this skill:

- Provide the Yingdao input-box value as exactly one physical-line Python expression. Never show a multiline Python expression for that field.
- Serialize structured inputs with inline import syntax, for example `__import__('json').dumps({"date": str(run_date), "shop": str(shop_name)}, ensure_ascii=False)`. Do not assume `json` was imported earlier and do not show a raw JavaScript/Python object as the action input.
- Parse the received string inside the webpage JavaScript. Support standard JSON strings, double-encoded JSON strings, and quoted scalar fallbacks when relevant.
- Provide result extraction as a one-line Python expression too, for example `__import__('json').loads(js_result).get("data", [])`.

Use conservative syntax in final scripts when compatibility is uncertain:

- Prefer `var` and normal functions.
- Avoid optional chaining, top-level `return`, bare IIFEs, and complex shell-escaped snippets.
- Avoid relying on multiple arguments to `page.evaluate`; pass one object when using Playwright/opencli runners.
- Decode GBK responses with `TextDecoder("gbk")` when response headers or preview text indicate GBK.
- For downloads, remember that webpage JavaScript cannot reliably write arbitrary local files. Prefer returning `downloadUrl` and `savePath`, then use a separate Yingdao "run/open" step such as PowerShell `Invoke-WebRequest` when the user does not want Yingdao's packaged download action.
- Keep every Yingdao "set variable" expression on one physical line; do not require helper functions or prior imports.
- For dynamic structured inputs, use the same one-line serialization pattern, for example `__import__('json').dumps({"jobId": jobId}, ensure_ascii=False)`.

Input normalization pattern:

```js
function cleanScalar(input) {
  var s = String(input || "").trim();
  try {
    if (s.length >= 2 && ((s[0] === '"' && s[s.length - 1] === '"') || (s[0] === "'" && s[s.length - 1] === "'"))) {
      s = JSON.parse(s);
    }
  } catch (e) {
    s = s.replace(/^["']|["']$/g, "");
  }
  return s;
}
```

## Final Answer

Be concise and include:

1. Confirmed endpoint/runtime/module and whether UI operation was used.
2. Required request fields and response paths.
3. The runnable Yingdao webpage JavaScript.
4. Live-page requirements such as "run on the logged-in page".
5. Any blocker or risk-control condition, if present.
