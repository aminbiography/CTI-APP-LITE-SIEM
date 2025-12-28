Live URL:  https://aminbiography.github.io/CTI-APP-LITE-SIEM/ 

---
 
# Windows OS CTI-LITE SIEM

## User Guide (SOC Analyst / Cybersecurity Analyst / CTI Associate)

### 1) What this tool is

**Windows 11 CTI-lite SIEM** is a lightweight, browser-based utility that helps you perform **initial CTI matching** against **exported Windows logs**. It is designed for:

* quick triage,
* IOC sweep across logs,
* rapid validation of suspicious indicators,
* generating a shareable JSON report.

### 2) What this tool is not

This tool is **not a full SIEM** and it does **not**:

* collect logs in real time,
* read Windows Event Logs directly (no live EVTX ingestion),
* run correlation rules like enterprise SIEMs,
* replace EDR or an incident response platform.

### 3) What you need before you start

You need:

* A Windows 11 system (or any system where you can export Windows logs).
* Exported logs in one of these formats:

  * **Windows Event Log XML** (recommended) exported using `wevtutil`
  * **JSON** (array or line-delimited JSON)
* CTI indicators in plain text lists:

  * IPs (one per line)
  * Domains (one per line)
  * URLs (one per line)
  * Hashes (MD5/SHA1/SHA256, one per line)

---

## 4) Step-by-step usage

### Step A - Export Windows logs (recommended approach)

This page cannot read `.evtx` directly, so export to XML first.

Example exports:

```text
wevtutil qe Security /c:2000 /rd:true /f:xml > security.xml
wevtutil qe Microsoft-Windows-Sysmon/Operational /c:2000 /rd:true /f:xml > sysmon.xml
```

Practical notes:

* `/c:2000` limits the number of events. Increase if needed.
* `/rd:true` reads newest events first.
* You can export other channels too (Defender, PowerShell, etc.).

---

### Step B - Load logs into the tool

1. Open the HTML page in your browser.
2. Under **1) Load logs**, click **Choose File** and select your exported XML or JSON file.
3. Set **Max events** (example: 3000).

   * For very large logs, start smaller for performance.
4. If using JSON, adjust **Text fields to scan** (example: `message, RenderedDescription, EventData, UserData`).
5. Click **Parse Logs**.
6. Confirm the header status updates to something like:

   * `Parsed XML: #### events` or `Parsed JSON: #### events`

If parsing fails:

* Verify the file format.
* Try exporting again.
* Open browser console (F12) to see parsing errors.

---

### Step C - Load your CTI feeds (offline lists)

1. Under **2) Load CTI feeds**, paste indicators into the correct boxes:

   * IPs
   * Domains
   * URLs
   * Hashes
2. Optional: click **Load sample feeds** (demo only).
3. Confirm the header shows:

   * `Feeds: X (ip:a domain:b url:c hash:d)`

Best practices:

* Keep feeds clean and one indicator per line.
* Use comment lines starting with `#` for notes (these are ignored).

---

### Step D - Run extraction + matching

1. Choose **Minimum severity**:

   * **HIGH**: focuses on URLs/hashes (most actionable)
   * **MEDIUM**: includes domains/IPs (more coverage)
   * **LOW**: broadest
2. Click **Run Matching**
3. Review:

   * KPI counters: Parsed events, Extracted IOCs, Matches, Unique matched IOCs
   * Findings table: severity, IOC, type, evidence snippet

How to read results:

* **HIGH matches** (hash/URL) often require immediate validation (process, file, network).
* **Domain/IP matches** may include false positives; validate via context and allowlisting.

---

### Step E - Search and triage results

Use **Search findings** to filter by:

* IOC value,
* channel,
* event ID,
* time,
* evidence text.

This is intended for fast triage (e.g., “show only this domain” or “show everything from Sysmon”).

---

### Step F - Export report for sharing

Click **Export Report JSON** to download:
`cti-lite-siem-report.json`

Use cases:

* attach to an incident ticket,
* share with a SOC lead,
* preserve the analysis snapshot.

---

## 5) Analyst operational guidance

* Treat matches as **leads**, not final verdicts.
* Validate with:

  * Sysmon event context (process, parent, command line),
  * Defender/EDR telemetry,
  * threat intel enrichment (VirusTotal, MISP, TIP, sandbox) via your approved workflow.
* Maintain allowlists to reduce noise (recommended future enhancement).

---

# Developer Guide

## How Developers Benefit and How to Extend It

### 1) Why this project is useful to developers

This project is valuable because it is:

* **portable**: single HTML file, deploy anywhere
* **auditable**: no hidden backend logic
* **safe for demonstrations**: no API keys required
* **fast to customize**: rules, parsing, UI, export format can be modified easily
* **training-friendly**: great for SOC training labs and detection engineering exercises

---

## 2) Architecture overview (developer perspective)

### Processing pipeline

1. **Input**: user uploads XML/JSON logs
2. **Parse**:

   * XML via `DOMParser`
   * JSON via `JSON.parse` or line-delimited parsing
3. **Normalize**: produce consistent internal event schema
4. **Extract IOCs**: regex-based extraction from event text fields
5. **Match**: set membership checks against offline feed sets
6. **Rank and filter**: severity heuristic + minimum threshold
7. **Render + export**: HTML table rendering + JSON export

Key data structures:

* `STATE.events`: normalized events
* `STATE.feeds`: `{ips, domains, urls, hashes}` as `Set`
* `STATE.findings`: matched results list
* `STATE.report`: export object

---

## 3) How to customize it (high-impact modifications)

### A) Improve IOC extraction quality

Current method is regex-only. Enhancements:

* stricter domain rules (reduce false positives)
* URL normalization (strip trailing punctuation)
* detect IPv6
* parse email IOCs
* add registry keys, file paths, mutexes (optional)

Where to modify:

* `extractIOCs(text)`
* regex constants

---

### B) Add allowlisting (recommended for SOC realism)

Allowlisting reduces noise and improves signal.

Implementation pattern:

* Create allowlist sets (domains, IP ranges, known safe hashes)
* Check allowlist **before** counting as a match

Where to modify:

* inside the matching loop in `btnRun` handler

---

### C) Add an event detail viewer (major usability upgrade)

Current table provides snippets only. Add:

* a clickable row
* a side drawer/modal
* show:

  * full event message
  * provider, computer
  * extracted IOCs
  * raw event JSON (for JSON input)

Where to implement:

* `renderFindings()` row click handler
* add a hidden `<div>` modal/drawer in HTML

---

### D) Add persistence (feeds and last-used settings)

Currently everything resets on refresh.

Add:

* `localStorage` for:

  * feeds textareas
  * maxEvents
  * jsonFields
  * minSeverity
* or use IndexedDB for larger storage

Where to implement:

* on input change: save
* on load: restore

---

### E) Add CTI enrichment safely (do not expose API keys)

Correct approach:

* create a local proxy backend (Node/Python)
* browser calls your proxy (no keys in client)
* proxy calls VirusTotal/MISP/TIP securely
* cache results

Do not:

* embed API keys in JavaScript
* call paid CTI APIs directly from browser

---

## 4) Performance improvements (developer roadmap)

If you increase event volume (e.g., 50k–200k), you will need:

* **Web Workers** for parsing and matching (prevent UI freezing)
* **Virtualized rendering** (render only visible rows)
* incremental matching updates (batch loop with `requestAnimationFrame`)

---

## 5) Developer deployment options

### Option A - GitHub Pages

* Upload `index.html`
* Enable GitHub Pages
* Use as a static site

### Option B - Internal SOC portal

* Host behind SSO/VPN
* Add security headers (CSP, no-referrer, nosniff)

### Option C - Offline SOC workstation

* Keep HTML on analyst workstation
* Run locally with `python -m http.server`

---

## 6) Security posture for developers

What is good already:

* no backend
* no external requests
* no credential exposure

What to watch:

* log files can be sensitive (hostnames, usernames, internal IPs)
* do not upload logs to untrusted public hosting environments
* if hosted publicly, implement strict CSP and no telemetry by default

---

## 7) Suggested “next version” features (practical)

* IOC allowlisting and suppression rules
* event detail drawer
* CSV export
* MITRE ATT&CK tagging (manual mapping rules)
* rule packs for:

  * Sysmon process creation
  * PowerShell ScriptBlock
  * Defender detections
* pluggable parsers per log source

---
