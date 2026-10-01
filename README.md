# Day 29: Scoped Application VAPT (DVWA Milestone Lab)

## 📌 Project Overview
This repository contains the technical walkthroughs, execution data streams, configuration findings, and manual exploit verifications compiled during **Day 29 of my Vulnerability Assessment and Penetration Testing (VAPT) Internship with TriosCyber** (in partnership with Ernith).

The primary objective of this project was to perform an end-to-end, multi-phase security assessment against a dedicated web application target. The methodology maps a realistic penetration testing lifecycle, combining layer-4 service fingerprinting via **Nmap**, path discovery fuzzing via **FFUF**, baseline configuration scanning via **Nikto**, and manual high-severity vulnerability exploitation via **Burp Suite Community Edition**.

---

## 🛠️ Lab Environment & Target Details
* **Target Application Endpoint:** Damn Vulnerable Web Application (`/dvwa/`) [54]
* **Target Host IP Address:** `192.168.145.134` [54]
* **Target MAC Address:** `00:0C:29:3B:0C:7F` (VMware Virtual Interface) [54]
* **Application Security Staging Level:** Low (`security=low`) [54]
* **Core Technical Suite:** `nmap`, `ffuf`, `nikto`, Burp Suite Repeater, Mozilla Firefox [54]

---

## 🚀 Step-by-Step Test Sequence & Technical Findings

### 1. Active Service Scanning (Nmap)
Executed target service version profiling (`-sV`) and active default script validation (`-sC`) restricted to active HTTP/S listeners on the target host:

```bash
nmap -sC -sV -p 80,443 192.168.145.134
```
[55]

#### 📋 Discovered Services & Network Properties:
* **Port `80/TCP` (Open):** Apache httpd `2.2.14` (Ubuntu) running `PHP/5.3.2` and trailing server modules (`mod_mono/2.4.3`, `mod_python/3.3.1`, OpenSSL `0.9.8k`, Phusion Passenger `4.0.38`).
* **Port `443/TCP` (Open):** Apache httpd `2.2.14` (Ubuntu) secure SSL layer handler.
* **Side-Channel Risk Vector:** Both web endpoints explicitly disclose support for the insecure `HTTP TRACE` connection method, exposing user sessions to Cross-Site Tracing (XST) vectors.
* **SSL Validation Data:** Cryptographic cert issued natively to `commonName=owaspbwa`.

---

### 2. Endpoint Content Discovery (FFUF)
Brute-forced application folder hierarchies within the authenticated web layout space utilizing the `common.txt` dictionary, explicitly passing valid low-level session parameters through the tracking string array:

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://192.168.145 -mc 200,301,302 -b "security=low; PHPSESSID=c8q05ruidd0b6otc507c15co0"
```
[56]

#### 📋 Discovered Application Paths:
* `/exec` (`301 Moved` | 257 B): Shell command execution processing portal.
* `/fi` (`301 Moved` | 255 B): Local file inclusion handler framework.
* `/upload` (`301 Moved` | 259 B): Filesystem storage and upload gateway interface.
* `/captcha` (`301 Moved` | 260 B): Insecure CAPTCHA confirmation routine.

---

### 3. Baseline Automated Web Configuration Scan (Nikto)
Executed a fast automated vulnerability check targeting the sub-application directory path to analyze configuration errors, security hygiene issues, and exposed reference folders:

```bash
nikto -h http://192.168.145 -o day29_nikto.txt
```
[57]
* **Key Findings:** Flagged a total absence of crucial security headers (`Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`), outdated software modules, and active unauthenticated directory indexing on `/dvwa/docs/` and `/dvwa/config/`.

---

### 4. Manual Test 1: OS Command Injection Verification
* **Target Endpoint Route:** `POST /dvwa/vulnerabilities/exec/` [57]
* **Vulnerable Parameter:** `ip` body parameter [57]
* **Procedure:** Intercepted the baseline request via Burp Proxy and transferred it to Burp Repeater. Appended a shell command separator character operator (`%3B`) to the input buffer to chain secondary system execution queries (`whoami` and `uname -a`):

```http
POST /dvwa/vulnerabilities/exec/ HTTP/1.1
Host: 192.168.145.134
Content-Type: application/x-www-form-urlencoded
Cookie: security=low; PHPSESSID=c8q05ruidd0b6otc507c15co0

ip=172.0.0.1%3B+whoami%3B+uname+-a&Submit=Submit
```
[57]
* **Observed Server Response:** `HTTP/1.1 200 OK`. The server executed the secondary logic commands and returned the output inside standard HTML `<pre>` tags, confirming complete shell access as user `www-data` on the host machine.

---

### 5. Manual Test 2: Reflected Cross-Site Scripting (XSS) Validation
* **Target Endpoint Route:** `GET /dvwa/vulnerabilities/xss_r/` [57]
* **Vulnerable Parameter:** `name` query parameter [57]
* **Procedure:** Loaded the input point inside Burp Repeater and substituted the standard alphanumeric string payload with an unencoded browser script execution token payload:

```http
GET /dvwa/vulnerabilities/xss_r/?name=%3Cscript%3Ealert%281%29%3C%2Fscript%3E HTTP/1.1
Host: 192.168.145.134
Cookie: security=low; PHPSESSID=c8q05ruidd0b6otc507c15co0
```
[58]
* **Observed Server Response:** `HTTP/1.1 200 OK`. The web application accepted the query string argument and reflected the literal characters `<script>alert(1)</script>` into the resulting DOM context completely unencoded. The browser parsed the text as raw layout commands and executed the popup script successfully.

---

## 📊 Scoped Assessment Summary Matrix

| Security Assessment Phase | Tooling Engine | Evaluation Objective | Core Technical Findings | Vulnerability Severity |
| :--- | :--- | :--- | :--- | :---: |
| **Port & Service Scanning** | `nmap` | Map perimeter services and active banner strings on ports 80/443. | Apache 2.2.14, PHP 5.3.2, and insecure `HTTP TRACE` methods active. | **Informational** [58] |
| **Endpoint Content Discovery** | `ffuf` | Brute-force directory paths to locate target modules. | Located endpoints `/exec/`, `/fi/`, `/upload/`, and `/captcha/`. | **Informational** [58] |
| **Automated Configuration Scan**| `nikto` | Sweep for application misconfigurations and directory indexing. | Confirmed missing security headers and exposed `/config/` index trees. | **Informational** [58] |
| **Manual Verification Test 1** | Burp Repeater | Exploit the input fields of the network command tool utility. | **Confirmed OS Command Injection:** Achieved full code execution as `www-data`. | 🔴 **High (RCE)** [59] |
| **Manual Verification Test 2** | Burp Repeater | Manipulate data parameters inside the dynamic text field. | **Confirmed Reflected XSS:** Input tags reflect into HTML context unencoded. | 🟡 **Medium (XSS)** [59] |

---

## 🛡️ Key Takeaways & Remediation Guidelines

### 1. Disable Legacy and High-Risk HTTP Protocols
Shut down potential Cross-Site Tracing (XST) vectors by disabling the `TRACE` method within your main server configurations. Update your core Apache environment properties to enforce strict protocol handling constraints:
```apache
TraceEnable Off
```
[59]

### 2. Remediate Remote Code Execution (RCE) Vectors
Web application backends must never pass unvalidated string arguments directly into operating system shell execution environments. Eliminate raw wrappers like `system()` or `shell_exec()`, and implement ironclad regex validations to ensure variable inputs conform strictly to clean data specifications:
```php
// Enforcing Strict Data Validation Constraints inside PHP
if (filter_var(\$ip, FILTER_VALIDATE_IP)) {
    // Process parameter safely following format validation
}
```

### 3. Mitigate Client-Side Script Injection (XSS)
Neutralize reflected scripting exploits by enforcing context-aware output encoding routines on all dynamic user parameters. Pass input fields through a native string sanitizer function (such as `htmlspecialchars()`) before letting strings map onto browser DOM interfaces:
```php
echo "Hello " . htmlspecialchars(\$_GET['name'], ENT_QUOTES, 'UTF-8');
```

---

## 📂 Repository Structure
```plaintext
.
├── README.md
├── screenshots/
│   ├── nmap_web_discovery.png
│   ├── ffuf_path_fuzzing.png
│   ├── nikto_raw_findings.png
│   ├── rce_exploit_repeater.png
│   └── reflected_xss_poc.png
└── outputs/
    ├── nmap_report.txt
    ├── ffuf_endpoints.txt
    └── nikto_vulnerabilities.txt
```

---

## 👤 Author
**Azeez Umar Opeyemi**
* 💼 **Role:** VAPT Intern at TriosCyber
* 📧 **Email:** umaropeyemiazeez@gmail.com
* 🐙 **GitHub:** [itzumaz](https://github.com/itzumaz)
* 🔗 **LinkedIn:** [Azeez Umar Opeyemi](https://www.linkedin.com/in/azeez-umar-opeyemi-201a433a4/)
