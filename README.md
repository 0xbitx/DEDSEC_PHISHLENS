
<p align="center">
  <img src="https://media1.giphy.com/media/v1.Y2lkPTc5MGI3NjExZGF1NGtlNHQ4eWlqb3RhamZhbzdndm05aWFvdnlqdHpxd2FxaWQ2dSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/PPUtKoa3dOW3cTNL4M/giphy.gif">
</p>


<h1 align="center">PHISHLENS</h1>
<p align="center"><code>Advanced Phishing Detection Engine — Multi-Layer Analysis, Penalty-Based Scoring, Automated Triage</code></p>

---

## Overview

PhishLens is an advanced phishing detection engine built for security operations. It ingests a suspect URL and performs automated triage across 16 independent detection layers spanning adversary infrastructure analysis, content forensics, statistical classification, and visual inspection. The tool was designed to counter the techniques observed in real phishing campaigns: URL shortener chains, `@` symbol redirection with Unicode obfuscation, IDN homograph attacks, credential harvesting forms posting to external domains, hidden iframes, and obfuscated JavaScript payloads.

Each layer produces an independent finding surfaced in the console, embedded in a structured PDF report, and exported as machine-readable JSON for downstream SIEM and SOAR ingestion. A weighted scoring engine combines the ML classifier with infrastructure signals, then applies a penalty system for concrete red flags such as `@` redirection attacks, URL shortener usage, login form exfiltration, and brand impersonation — driving the final score toward zero for confirmed phishing infrastructure.

## Detection Layers

### Network & Infrastructure

**Redirect Chain Tracing**
Each redirect hop is followed and reported. Chains longer than two hops are flagged as adversary infrastructure commonly routes victims through multiple compromised domains before landing on the final phishing page.

**Homograph & Typosquatting Detection**
The domain is checked for IDN punycode encoding and compared against a corpus of over 30 high value brand names. Domains containing brand strings that do not match the official domain are flagged as typosquatting attempts.

**URL @ Redirection Detection**
The `@` symbol in a URL causes browsers to treat everything before it as authentication credentials and silently ignore it, routing the victim to whatever domain appears after the `@`. Attackers exploit this by placing a legitimate looking domain before the `@` (e.g. `https://facebook.com/path@evil.com`). This layer extracts both the decoy domain and the real destination, and also detects Unicode look-alike characters (such as U+29F8 ⧸ mimicking `/`, U+0294 ʔ mimicking `?`, U+A78A ꞊ mimicking `=`) used to fabricate convincing URL paths that are actually part of the ignored userinfo portion.

**URL Shortener Expansion**
When a URL shortener service (t.ly, bit.ly, is.gd, cutt.ly, and 25+ others) is detected, the engine performs a HEAD request to extract the `Location` header and expands the shortened link to its real destination. All subsequent domain-based lookups target the expanded phishing infrastructure rather than the intermediary shortener service. This layer runs before any other analysis to ensure every downstream check operates on the true adversary host.

**Domain Reputation (VirusTotal)**
The domain is submitted to the VirusTotal API v3. The `last_analysis_stats` object is parsed and any vendor flagging the domain as malicious is surfaced. Missing or invalid API keys are reported explicitly rather than producing a silent failure.

**SSL Certificate Inspection**
A direct TLS handshake extracts the full certificate chain for offline analysis. The issuer is identified (LetsEncrypt, DigiCert, self-signed, etc.), certificate age is calculated in days, Subject Alternative Names are examined for unrelated domains, and certificates issued within the last 7 days are flagged as high risk. No external CT log dependency required.

### Content & Structure

**SSL/TLS Certificate Validation**
A TLS handshake is performed against port 443. The peer certificate's `notAfter` field is compared against system time. Expired or unverifiable certificates are treated as risk indicators.

**URL Keyword Inspection**
The URL path, query string, and `@` decoy portion are scanned against a corpus of over 90 high signal keywords: credential harvesting triggers, urgency pressure phrases, financial lure terms, and brand impersonation targets. The domain name itself is excluded from scanning to prevent false positives on legitimate sites.

**Favicon Hash**
The site's favicon is downloaded and its SHA256 hash is computed. This enables comparison against known legitimate favicon databases and can detect cloned assets hosted on adversary infrastructure.

**Page Content Analysis**
The rendered DOM is inspected for phishing indicators: login forms posting credentials to external domains, hidden iframes loading third party content, obfuscated JavaScript patterns (`eval`, `fromCharCode`, `atob`), and brand name references on pages with login forms that do not match the hosting domain.

**Domain Registration Analysis**
WHOIS lookup retrieves the domain creation timestamp. Domains under 365 days old are flagged as elevated risk since adversary operated domains are overwhelmingly short lived.

### Classification & Scoring

**Machine Learning Classification**
A custom-trained XGBoost classifier (`phishing_model_by_0xbit.model`) extracts 114 structural features from the URL across the domain, path, query string, and filename components. The model was trained on a curated phishing dataset using gradient-boosted decision trees with an 80/20 train/test split. Features include character frequency distributions, IP-in-hostname detection, TLD analysis, URL shortener detection, DNS TTL values, MX record counts, WHOIS domain age in days, measured HTTP response time, and structural anomaly scoring. The model outputs a binary classification with a confidence probability that carries the largest weight in the scoring engine.

**Penalty-Based Safety Scoring**
The scoring engine combines two mechanisms. First, a weighted base score is computed from the ML classifier (60%) and four infrastructure signals — domain reputation, SSL validity, keyword detection, and domain age — at 10% each. When the VirusTotal API key is not configured, its weight is dynamically redistributed to the ML model rather than counting as a flat penalty. Then a penalty system deducts from the base for concrete red flags: `@` redirection (-15%), URL shortener usage (-10%), login forms posting credentials to external domains (-10%), brand impersonation in page content (-5%), and IDN homograph attacks (-8%). This dual approach ensures that a phishing page with valid SSL and an unknown reputation (which would otherwise score deceptively high) is driven toward zero when structural attack techniques are confirmed. Legitimate domains with no penalty triggers retain their full weighted score.

### Output & Automation

**PDF Report Generation**
A structured five-section PDF is produced containing an executive summary with pass/fail counts, domain information tables, categorized detection findings with descriptions and status badges, a redirect flow map, an embedded full-page screenshot, and a scoring methodology breakdown.

**JSON Export**
All analysis results are exported as structured JSON alongside the PDF. The schema is designed for ingestion into SOAR playbooks, SIEM dashboards, or custom analysis pipelines.

**Batch Processing**
A file path can be provided instead of a single URL. The tool ingests newline-delimited URL lists or CSV exports from email gateways and processes each entry sequentially, producing individual reports for each.

**Visual Triage**
Selenium WebDriver renders the target page at full resolution and captures a screenshot saved to `page-screenshots/`. The screenshot is embedded directly in the PDF report.

## Installation

```bash
git clone https://github.com/0xbitx/DEDSEC_PHISHLENS.git
cd DEDSEC_PHISHLENS
pip3 install -r requirements.txt
```

Obtain a VirusTotal API key from https://www.virustotal.com and write it to the `virustotal.api` file, replacing the placeholder value.

```bash
chmod +x dedsec_phishlens
./dedsec_phishlens
```

## Dependencies

| Package           | Purpose                          |
|-------------------|----------------------------------|
| requests          | VirusTotal API & HTTP communication |
| selenium          | Headless browser automation      |
| webdriver-manager | ChromeDriver binary management   |
| python-whois      | Domain registration lookups      |
| joblib            | ML model serialization           |
| pandas            | Feature dataframe construction   |
| tabulate          | Console output formatting        |
| dnspython         | DNS MX, SPF, and TTL resolution  |
| fpdf2             | PDF report generation            |

Chrome or Chromium must be installed on the host system for the screenshot capture module. The WebDriver binary is managed automatically at runtime.

## Usage

Launch the tool and provide a URL at the prompt. The engine runs all 16 detection layers sequentially and displays each finding inline. A final weighted safety percentage is printed along with paths to the PDF report, JSON export, and captured screenshot.

```bash
./dedsec_phishlens
```

To batch process multiple URLs, provide a file path instead of a URL. The tool accepts newline-delimited URL lists or single-column CSV files and produces individual reports for each entry.

```bash
./dedsec_phishlens
# At the prompt, enter: urls.txt
```

All screenshots persist in `page-screenshots/` and reports land in `reports/`, named by domain and timestamp for audit trail purposes.

## Supported Platforms

The tool has been verified on the following distributions:

* Kali Linux
* Parrot OS
* Ubuntu

## Project Structure

```
DEDSEC_PHISHING_DETECTION/
├── dedsec_phishlens                Main analysis engine
├── phishing_model_by_0xbit.model   Custom-trained XGBoost classifier
├── virustotal.api                  VirusTotal API key (user provided)
├── requirements.txt                Python dependencies
├── page-screenshots/               Captured page renders
└── reports/                        PDF reports and JSON exports
```

## Threat Model

PhishLens addresses the initial triage phase of the phishing incident response lifecycle with an emphasis on the techniques most commonly observed in active campaigns: `@` redirection with Unicode path fabrication, URL shortener chains hiding adversary infrastructure, Cloudflare-proxied phishing pages with valid SSL certificates, and credential harvesting forms embedded in brand-impersonating landing pages. The penalty-based scoring model is specifically designed to counter the SSL blind spot — a phishing page behind Cloudflare with a valid certificate would score 40% or higher under a simple weighted model, but the penalty system drives it toward single digits when structural attack techniques are confirmed. A URL that bypasses one or two layers will still be caught by the remaining signals.

## Use in SOC Workflows

This tool fits into a tier 1 analysis workflow where URLs arrive from user reports, email gateway alerts, or threat intelligence feeds. An analyst can batch process indicators through PhishLens and triage results by weighted safety score. Low scoring URLs can be escalated for deeper analysis while high scoring URLs can be dismissed with supporting evidence logged from the tool's output.

Automated integration into a SOAR playbook is feasible by calling the underlying Python class directly and consuming the structured output programmatically.

## Limitations

This tool is an analysis aid, not a definitive verdict. Sophisticated phishing campaigns may employ techniques such as cloaking, geofencing, or CAPTCHA gating that prevent automated screenshot capture and may evade keyword or age based detection. Analysts should treat the weighted score as one input among many in their investigation process and always apply contextual judgment.

| TOOL BANNER |
|----------|
| ![image-analysis](https://github.com/user-attachments/assets/cb5cff04-6ce9-4751-82b8-bdd161e92084) |

## Detection Gallery

Real-world URL analyses performed by PhishLens. Each entry shows the target URL and the full-page screenshot captured during automated triage, exactly as an analyst would review it during an investigation.

---

### 1. GitHub (Legitimate Domain)

| Analysis |
|----------|
| URL: `https://github.com` |
| ![image-analysis](https://github.com/user-attachments/assets/792c7a59-c6f7-4b1f-8ea2-1cf89fe65fe5) |
| Report |
| [report-github.com-20260723065712.pdf](https://github.com/user-attachments/files/30289856/report-github.com-20260723065712.pdf) |
| Json Report |
| [report-github.com-20260723065713.json](https://github.com/user-attachments/files/30289851/report-github.com-20260723065713.json) |

---

### 2. Google (Legitimate Domain)

| Analysis |
|----------|
| URL: `https://google.com` |
| ![image-analysis](https://github.com/user-attachments/assets/8788d7a9-cc7c-4602-819e-78240708c721) |
| Report |
| [report-google.com-20260723070032.pdf](https://github.com/user-attachments/files/30289833/report-google.com-20260723070032.pdf) |
| Json Report |
| [report-google.com-20260723070032.json](https://github.com/user-attachments/files/30289835/report-google.com-20260723070032.json) |


---

### 3. Facebook Login Page (Legitimate Domain)

| Analysis |
|----------|
| URL: `https://facebook.com` |
| ![image-analysis](https://github.com/user-attachments/assets/20b2c055-0cbc-4f91-9424-92333673f128) |
| Report |
| [report-facebook.com-20260723070133.pdf](https://github.com/user-attachments/files/30289811/report-facebook.com-20260723070133.pdf) |
| Json Report |
| [report-facebook.com-20260723070133.json](https://github.com/user-attachments/files/30289815/report-facebook.com-20260723070133.json) |

---

### 4. Actual Phishing Page (Facebook Method 1)

| Analysis |
|----------|
| URL: `https://www.facebook.com⧸james⧸idʔ@prepare-primary-favors-attitudes.trycloudflare.com` |
| ![image-analysis](https://github.com/user-attachments/assets/ff77eee9-e3c6-4b95-88ac-4cba99fc8b12) |
| ![image-analysis](https://github.com/user-attachments/assets/1bf2707c-e38d-4059-8ff2-c8a303e4d5bb) |
| My Phishing Tool |
| ![image-analysis](https://github.com/user-attachments/assets/15ac761c-1d8d-45c5-9955-66ee095db27a) |
| Report |
| [report-prepare-primary-favors-attitudes.trycloudflare.com-20260723070512.pdf](https://github.com/user-attachments/files/30289760/report-prepare-primary-favors-attitudes.trycloudflare.com-20260723070512.pdf) |
| Json Report |
| [report-prepare-primary-favors-attitudes.trycloudflare.com-20260723070512.json](https://github.com/user-attachments/files/30289762/report-prepare-primary-favors-attitudes.trycloudflare.com-20260723070512.json) |

---

### 5. Actual Phishing Page (Facebook Method 2)

| Analysis |
|----------|
| URL: `https://www.facebook.com⧸james⧸idʔ@t.ly/Rz9Jn` |
| ![image-analysis](https://github.com/user-attachments/assets/f8f645f5-e5ca-43d9-8608-a90b03bcec5c) |
| ![image-analysis](https://github.com/user-attachments/assets/5202a613-c5cd-4887-8e32-39153f63c8de) |
| Report |
| [report-prepare-primary-favors-attitudes.trycloudflare.com-20260723070757.pdf](https://github.com/user-attachments/files/30289734/report-prepare-primary-favors-attitudes.trycloudflare.com-20260723070757.pdf) |
| Json Report |
| [report-prepare-primary-favors-attitudes.trycloudflare.com-20260723070757.json](https://github.com/user-attachments/files/30289737/report-prepare-primary-favors-attitudes.trycloudflare.com-20260723070757.json) |

### 6. Actual Phishing Page (Google Method 1)

| Analysis |
|----------|
| URL: `https://google.com⧸accountʔ@commands-may-opposed-leu.trycloudflare.com` |
| ![image-analysis](https://github.com/user-attachments/assets/22650366-5795-4cad-8400-366e45580ad1) |
| ![image-analysis](https://github.com/user-attachments/assets/888a3aac-3729-4f51-bb97-126bd9a37b04) |
| My Phishing Tool |
| ![image-analysis](https://github.com/user-attachments/assets/2aad73cc-6475-41ba-a01b-645a1317f899) |
| Report |
| [report-commands-may-opposed-leu.trycloudflare.com-20260723110202.pdf](https://github.com/user-attachments/files/30290843/report-commands-may-opposed-leu.trycloudflare.com-20260723110202.pdf) |
| Json Report |
| [report-commands-may-opposed-leu.trycloudflare.com-20260723110203.json](https://github.com/user-attachments/files/30290846/report-commands-may-opposed-leu.trycloudflare.com-20260723110203.json) |

---

### 7. Actual Phishing Page (Google Method 2)

| Analysis |
|----------|
| URL: `https://google.com⧸accountʔ@t.ly/4l6sF` |
| ![image-analysis](https://github.com/user-attachments/assets/31cd3ccf-ce46-4fab-bb01-92a6abc9d776) |
| ![image-analysis](https://github.com/user-attachments/assets/6dfd2348-88cc-4c91-880b-9cf349c3b641) |
| Report |
| [report-commands-may-opposed-leu.trycloudflare.com-20260723105655.pdf](https://github.com/user-attachments/files/30290771/report-commands-may-opposed-leu.trycloudflare.com-20260723105655.pdf) |
| Json Report |
| [report-commands-may-opposed-leu.trycloudflare.com-20260723105655.json](https://github.com/user-attachments/files/30290776/report-commands-may-opposed-leu.trycloudflare.com-20260723105655.json) |

---

## License & Disclaimer

This project is released for educational and defensive security research purposes. The author assumes no liability for misuse or for decisions made solely on the basis of this tool's output. Use responsibly and in accordance with applicable laws and organizational policies.
