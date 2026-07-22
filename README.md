# DEDSEC PHISHLENS

**Automated Phishing URL Analysis & Triage Platform**

---

## Overview

PhishLens is a phishing detection and analysis engine built to support security operations workflows. It ingests a URL and performs automated triage across five independent detection layers before producing a weighted safety verdict. The tool is designed to mirror the decision making process a SOC analyst follows when investigating a potentially malicious link: check threat intelligence, validate the TLS handshake, inspect the URL structure for social engineering indicators, assess domain registration history, and capture a visual reference of the landing page.

Each detection layer produces its own finding and the engine aggregates them into a single percentage based confidence score using equally weighted criteria. This makes PhishLens suitable both for manual investigation and as a decision support component in a broader phishing analysis pipeline.

## Detection Layers

**Threat Intelligence Lookup**
The domain is submitted to the VirusTotal API v3. The response is parsed for the `last_analysis_stats` object and any vendor flagging the domain as malicious is surfaced directly in the output. A clean reputation contributes positively to the final safety score.

**TLS Certificate Validation**
A TLS handshake is initiated against port 443 of the target domain. The peer certificate is extracted and its `notAfter` field is compared against the current system time. An expired or unverifiable certificate is treated as a risk indicator consistent with adversary infrastructure that is stood up and abandoned quickly.

**URL Keyword Inspection**
The full URL string is scanned against a corpus of over 90 high signal keywords commonly observed in phishing campaigns. These include credential harvesting triggers (`login`, `verify`, `password`, `confirm your identity`), urgency pressure phrases (`act now`, `limited time`, `final notice`), financial lure terms (`invoice`, `refund`, `payment declined`), and brand impersonation targets (`paypal`, `amazon`, `netflix`, `microsoft`). Any matches are reported with the specific tokens that fired.

**Domain Registration Analysis**
A WHOIS lookup retrieves the domain creation timestamp. Domains registered within the last 365 days are flagged as elevated risk. Adversary operated domains used in phishing campaigns are overwhelmingly short lived, making registration recency a high value signal in triage.

**Machine Learning Classification**
A pre-trained model (`phishing_model_by_0xbit.model`) extracts 114 structural features from the URL, decomposed across the domain, path, query string, and filename components. Features include character frequency distributions, presence of IP addresses in the host portion, TLD analysis, URL shortening service detection, and structural anomaly scoring. The model outputs a binary classification with a confidence probability that feeds directly into the weighted scoring engine.

**Weighted Safety Scoring**
Each of the five detection layers carries equal weight (20%) in the final calculation. Individual layer outcomes are binarized (safe or unsafe), multiplied by their weight, and summed to produce a percentage. This provides a human readable verdict that reflects consensus across independent detection methods rather than relying on any single signal.

**Visual Triage**
Selenium WebDriver renders the target page and captures a full resolution screenshot saved to the `page-screenshots/` directory. This allows the analyst to visually inspect the landing page for brand impersonation, credential harvesting forms, or other indicators that automated detection may miss.

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
| requests          | VirusTotal API communication     |
| selenium          | Headless browser automation      |
| webdriver-manager | ChromeDriver binary management   |
| python-whois      | Domain registration lookups      |
| joblib            | ML model serialization           |
| pandas            | Feature dataframe construction   |
| tabulate          | Console output formatting        |

Chrome or Chromium must be installed on the host system for the screenshot capture module. The WebDriver binary is managed automatically at runtime.

## Supported Platforms

The tool has been verified on the following distributions:

* Kali Linux
* Parrot OS
* Ubuntu


## Threat Model

PhishLens addresses the initial triage phase of the phishing incident response lifecycle. It is designed to provide rapid, multi-signal analysis of a suspect URL before an analyst commits time to deeper investigation or before the URL is escalated for takedown. The weighted scoring model reduces false positives by requiring consensus across independent detection methods. A URL that bypasses one or two layers (for example, a phishing page served over HTTPS with a valid certificate) will still be caught by the remaining layers if other indicators are present.

## Use in SOC Workflows

This tool fits into a tier 1 analysis workflow where URLs arrive from user reports, email gateway alerts, or threat intelligence feeds. An analyst can batch process indicators through PhishLens and triage results by weighted safety score. Low scoring URLs can be escalated for deeper analysis while high scoring URLs can be dismissed with supporting evidence logged from the tool's output.

Automated integration into a SOAR playbook is feasible by calling the underlying Python class directly and consuming the structured output programmatically.

## Limitations

This tool is an analysis aid, not a definitive verdict. Sophisticated phishing campaigns may employ techniques such as cloaking, geofencing, or CAPTCHA gating that prevent automated screenshot capture and may evade keyword or age based detection. Analysts should treat the weighted score as one input among many in their investigation process and always apply contextual judgment.

## License & Disclaimer

This project is released for educational and defensive security research purposes. The author assumes no liability for misuse or for decisions made solely on the basis of this tool's output. Use responsibly and in accordance with applicable laws and organizational policies.
