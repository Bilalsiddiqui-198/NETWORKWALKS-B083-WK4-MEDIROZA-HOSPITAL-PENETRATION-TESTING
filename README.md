# NETWORKWALKS-B083-WK4-MEDIROZA-HOSPITAL-PENETRATION-TESTING

# 🏥 Mediroza General Hospital — Penetration Testing Report

**Full Black-Box Web Application Security Assessment**

![Skill](https://img.shields.io/badge/Skill-Cybersecurity-404040?style=flat-square&labelColor=C00000)
![Engagement](https://img.shields.io/badge/Engagement-Black--box-0070C0?style=flat-square&labelColor=000000)
![Target](https://img.shields.io/badge/Target-medirozahospital.com-E87500?style=flat-square&labelColor=000000)
![Severity](https://img.shields.io/badge/Highest%20Risk-Critical-C00000?style=flat-square&labelColor=000000)
![Status](https://img.shields.io/badge/Status-Complete-238F89?style=flat-square&labelColor=000000)

**Target Domain:** https://medirozahospital.com
**Engagement Window:**  30/9 – 2/10, 2026
**Overall Risk Rating:** 🔴 **CRITICAL** (Immediate Remediation Required)

---

## ⚠️ Liability Disclaimer

This penetration testing report and all associated activities were performed strictly within an authorized scope, on a training environment for which written permission was granted by Networkwalks. The materials, findings, and methodologies detailed herein are provided solely for educational and authorized training purposes as part of the Networkwalks Cybersecurity Internship (Batch B083).

Any unauthorized use, reproduction, or execution of these techniques outside the permitted scope is strictly prohibited. Neither the author nor Networkwalks assumes any liability or responsibility for misuse of this information; all actions and consequences remain the sole responsibility of the user. All patient, staff, and shareholder records referenced are **synthetic training data**.

This engagement was structured into **4 milestones**: (M1) gaining initial access, (M2) cracking the retrieved files' encryption, (M3) uncovering a critical server-side data exposure, and (M4) this detailed report.

---

## 📋 Table of Contents

1. Executive Summary
   - 1.1 Summary of Assessment Findings
2. Scope & Methodology
   - 2.1 Rules of Engagement & Constraints
   - 2.2 Tools & Techniques
3. Technical Findings & Proof of Exploitation
   - 3.1 Milestone 1 — Initial Access & Retrieving the PDF Reports
   - 3.2 Milestone 2 — Cracking the Encryption on All 3 Files
   - 3.3 Milestone 3 — Critical Data Exposure on the Client Server
4. Risk Rating
5. Recommendations and Remediation
6. Tools & Resources
7. Security & Ethical Use
8. Author & Project Information

---

## 1️⃣ Executive Summary

A black-box web application penetration test was conducted against Mediroza General Hospital (`medirozahospital.com`) as part of Week 4 of the Networkwalks Cybersecurity Internship. The objective was to evaluate the security posture of the hospital's web infrastructure and demonstrate the real-world impact of any vulnerabilities discovered, including exposure of confidential patient and corporate data.

The assessment uncovered a severe, chained exploitation path:

- An **authentication bypass via SQL Injection** allowed unauthorized access to the internal Patient Portal, exposing confidential patient lab reports.
- **Metadata inspection** of the decrypted PDF reports revealed a reference to an exposed legacy server directory (`/old/`).
- This exposed directory contained a full SQL database backup (`mediroza_db_backup_2019.sql`), directly leaking **30 staff salary records** and **10 shareholder equity records**.

### 1.1 Summary of Assessment Findings

| Milestone / Area | Vulnerability Identified | Risk Level | Impact |
|---|---|---|---|
| M1: Initial Access | SQL Injection → Authentication Bypass | 🔴 **Critical** | Full bypass of authentication; access to restricted patient portal and lab reports |
| M2: Data Extraction | Weak/Crackable PDF Password Protection | 🟠 **High** | Offline recovery of encrypted PDF lab reports, exposing protected health information |
| M3: Data Exposure | Publicly Accessible SQL Database Backup (`/old/`) | 🔴 **Critical** | Direct leak of 30 staff salaries and 10 shareholder equity details |
| M3: Information Leak | Sensitive Metadata in Patient PDFs | 🟡 **Medium** | Exposed internal software version and legacy directory references, aiding further compromise |

---

## 2️⃣ Scope & Methodology

### 2.1 Rules of Engagement & Constraints

- **Authorized Domain Only:** Testing was confined strictly to `https://medirozahospital.com`.
- **Prohibited Actions:** Denial-of-Service (DoS), social engineering, and destructive actions against the target were strictly prohibited.
- **Confidentiality Protocol:** Sensitive data extracted during testing (patient data, salaries, shareholder data) is referenced at a summary level only in this report, without reproducing full sensitive values.

### 2.2 Tools & Techniques

| Tool / Command | Purpose & Technical Context |
|---|---|
| `whois` / `nslookup` | Domain registration and DNS resolution |
| `wafw00f` | Web Application Firewall fingerprinting |
| `gobuster` | Directory/route discovery (`/patient/`, `/staff/`, `/old/`) |
| Manual SQL Injection testing | Authentication bypass via the login form |
| Networkwalks Hash Calculator | Extraction of `$pdf$` encryption hashes from protected PDFs |
| Networkwalks Password Cracker | Dictionary-based password recovery against extracted hashes |
| `exiftool` | Extraction and inspection of metadata from retrieved PDF reports |

---

## 3️⃣ Technical Findings & Proof of Exploitation

### 3.1 Milestone 1 — Initial Access & Retrieving the PDF Reports

**Goal:** Attack the website and find the 3 confidential PDF lab reports of patients.
**Permission:** ✅ Written Permission Granted

#### Step 1. Reconnaissance

**WHOIS Lookup:**

![WHOIS Lookup](3-whois_1.png)
![WHOIS Lookup — Registrant Info](4-whois_2.png)

**Findings:** Domain is registered through **NameCheap, Inc.**, created on 14 Aug 2026. Registrant contact details are hidden behind **WHOIS privacy protection** ("Withheld for Privacy LLC"), so no real owner details are exposed publicly.

---

**Nslookup:**

![Nslookup](1-nslookup.png)

**Findings:** The domain resolves to IP address **199.188.201.16** — this is the server hosting the target application.

---

**Wafw00f (WAF Detection):**

![Wafw00f Scan](2-wafw00f.png)

**Findings:** **No Web Application Firewall detected.** This means requests to the application — including malicious ones — are not being filtered by a WAF layer, which increases how exploitable later vulnerabilities would be.

---

**Gobuster (Directory Discovery):**

![Gobuster Scan 1](5-gobuster_1.png)
![Gobuster Scan 2](6-gobuster_2.png)

**Findings:** Several directories and files were discovered, including `robots.txt`, `/staff/`, `/old/`, `/cgi-bin/`, and `wp-admin`. The `/old/` and `/staff/` paths stood out as unusual and worth investigating further in later milestones.

---

**robots.txt:**

![robots.txt](24-robots.txt.png)

**Findings:** `robots.txt` explicitly disallows `/patient/`, `/staff/`, and `/old/`. Since this file is publicly readable by anyone (not just search engines), it effectively confirms these paths exist and hints at where sensitive content might be.

---

#### Step 2. Analysing the Login Behaviour

Submitting a random username returned **"Username not found,"** while submitting a guessed username (`admin`) with a wrong password returned a different message, **"Incorrect password."**

![Invalid Username](7-username_not_found.png)
![Valid Username, Wrong Password](8-incorrect_password.png)

**Observation:** The application responds differently depending on whether a username exists or not. This isn't something every website does — it depends entirely on how that specific application was built — but in this case, it meant the existence of the `admin` account could be confirmed just by observing the site's response behaviour, without needing any password.

---

#### Step 3. Testing for SQL Injection

The payload `' or 1=1--` was entered in the username field to test how the application handled unexpected input. The application returned a raw, unhandled MySQL error message.

![SQLi Error Confirmed](9-login_vulnerable_to_SQLi.png)

**Findings:** This confirmed the login query was vulnerable to **SQL Injection** — user input was being inserted directly into a SQL query without proper sanitization.

---

#### Step 4. Exploiting the SQL Injection — Authentication Bypass

The payload `admin'--` was submitted in the username field. This comments out the rest of the SQL query (including the password check), bypassing authentication entirely.

![SQLi Payload Used](10-SQLi.png)

The application granted access without a valid password.

![Successful Unauthorized Login](11-succesfully_login.png)

**Findings:** Full **authentication bypass** achieved via SQL Injection, logging in as `admin` without knowing the real password.

---

#### ✅ Milestone 1 Deliverable: Proof of Access

Once inside, the dashboard displayed **3 confidential, password-protected PDF lab reports** belonging to named patients — exactly the target of this milestone.

---

### 3.2 Milestone 2 — Cracking the Encryption on All 3 Files

**Goal:** Crack the encryption on all 3 retrieved files.
**Permission:** ✅ Written Permission Granted

Each PDF was password-protected, so a different cracking approach was required for each — not every file cracked the same way.

**Report 1**

![Hash Extracted — Report 1](12-hash-of-report_1.png)
![Password Recovered — Report 1](13-password-of-report_1.png)

**Findings:** Hash extracted via the Networkwalks Hash Calculator and successfully cracked using the Networkwalks Password Cracker.

**Report 2**

![Hash Extracted — Report 2](14-hash-of-report_2.png)
![Password Recovered — Report 2](15-password-of-report_2.png)

**Findings:** Same method applied successfully.

**Report 3**

![Hash Extracted — Report 3](16-hash-of-report_3.png)
![Password Recovered — Report 3](17-password-of-report_3.png)

**Findings:** This file required an additional/different attempt before the correct password was recovered, confirming that a single wordlist/approach does not always work for every file.

#### ✅ Milestone 2 Deliverable: Recovered Contents

All 3 PDFs were successfully unlocked and opened:

![Report 1 Opened](18-Report_1.png)
![Report 2 Opened](19-report_2.png)
![Report 3 Opened](20-report_3.png)

---

### 3.3 Milestone 3 — Critical Data Exposure on the Client Server

**Goal:** Find the critical data exposure on the client server — specifically, staff salaries and shareholder details.
**Permission:** ✅ Written Permission Granted

#### Step 1. Examining File Properties (Metadata)

Beyond just reading the unlocked PDFs, their metadata was inspected using `exiftool`.

![PDF Metadata — Report 1](21-metadata_report_1.png)
![PDF Metadata — Report 2](22-metadata_report_2.png)
![PDF Metadata — Report 3](23-metadata_report_3.png)

**Findings:** Metadata for the first two reports revealed the backend software in use — **Mediroza CMS 1.4.2** — along with internal authorship details. Report 3's metadata, however, contained something far more significant: a **Comments field** left behind by an internal user (`j.malik`), reading:

> *"DB backup moved to /old before site migration, do not delete"*

This comment directly revealed the existence and location of a database backup, pointing straight to the `/old/` directory — the same path already flagged earlier in the Gobuster scan and `robots.txt`.

---

#### Step 2. Investigating the `/old/` Directory

Acting on the clue found in Report 3's metadata, `https://medirozahospital.com/old/` was accessed directly. This confirmed that **directory listing was enabled**, exposing a file: `mediroza_db_backup_2019.sql`.

![Old Directory Listing Exposed](25-old%20directory.png)

**Findings:** A full, unauthenticated database backup was publicly downloadable — this is the critical server-side exposure this milestone was targeting, discovered as a direct result of the metadata comment left in Report 3.

#### Step 3. Analysing the Database Backup

The downloaded `.sql` file was opened and inspected. It contained two tables:

**`staff` table** — 30 employee records (name, job title, department, email, phone, national ID, monthly salary):

![Staff Table Exposed](26-staff_data.png)

**`shareholders` table** — ownership percentage, shares held, and share class:

![Shareholders Table Exposed](27-shareholders_data.png)


#### ✅ Milestone 3 Deliverable: Documented Evidence

Both the employee salary data and shareholder details were successfully located and documented, fulfilling both tasks of this milestone.

---

## 4️⃣ Risk Rating

| # | Finding | Risk Rating | Justification |
|---|---|---|---|
| 1 | SQL Injection → Authentication Bypass | 🔴 **Critical** | Full account takeover without valid credentials; root cause of the entire attack chain |
| 2 | Exposed Database Backup (`/old/`) | 🔴 **Critical** | Unauthenticated access to a full database backup |
| 3 | Sensitive Staff & Shareholder Data Exposure | 🔴 **Critical** | Exposes PII (national IDs, salaries) and confidential ownership data |
| 4 | Unauthorized Access to Patient Reports | 🔴 **Critical** | Confidential medical data accessed without proper authorization |
| 5 | Weak/Crackable PDF Password Protection | 🟠 **High** | Sensitive documents protected by easily recoverable passwords |
| 6 | Inconsistent Login Error Messages | 🟡 **Medium** | Reveals whether a username exists via differing application responses |
| 7 | PDF Metadata / Legacy Path Disclosure | 🟡 **Medium** | Leaks internal software version and legacy directory structure |
| 8 | Verbose SQL Error Messages | 🟡 **Medium** | Confirms injection points and reveals backend database details |
| 9 | robots.txt Discloses Sensitive Paths | 🟢 **Low** | Publicly readable file unintentionally maps out sensitive directories |
| 10 | No WAF Detected | 🟢 **Low** | Increases exploitability of other findings but is not a flaw on its own |

---

## 5️⃣ Recommendations and Remediation

| Finding | Recommended Fix |
|---|---|
| SQL Injection | Use parameterized queries / prepared statements for all database interactions |
| Exposed Database Backup | Store backups outside the web root, encrypted, with restricted access |
| Directory Listing Enabled | Disable directory listing on all production web directories |
| Unauthorized Access to Reports | Implement proper server-side authorization checks per user/record |
| Weak PDF Passwords | Use strong, randomly generated passwords, or secure authenticated download links instead |
| Inconsistent Error Messages | Return a single, generic error message for both invalid usernames and wrong passwords |
| PDF Metadata Disclosure | Strip unnecessary metadata before distributing documents |
| Verbose Error Messages | Disable detailed database error output in production |
| robots.txt Disclosure | Avoid listing sensitive directory names in `robots.txt` |
| No WAF | Deploy a Web Application Firewall for defense-in-depth |

---

## 📦 Milestone 4 Deliverable

This document **is** the Milestone 4 deliverable, a complete professional penetration testing report covering the Executive Summary, Scope & Methodology, Findings & Proof of Exploitation, Risk Rating, and Recommendations for every milestone of this engagement.

---

## 🛠️ Tools & Resources

- **Gobuster:** https://github.com/OJ/gobuster
- **Wafw00f:** https://github.com/EnableSecurity/wafw00f
- **ExifTool:** https://exiftool.org/
- **Networkwalks Hash Calculator:** https://networkwalks.com/hash-calculator/
- **Networkwalks Password Cracker:** https://networkwalks.com/password-cracker/

---

## 🔐 Security & Ethical Use

This assessment was performed strictly within the authorized scope of the Networkwalks Cybersecurity Internship program. All patient, staff, and shareholder data referenced is synthetic training data created for this exercise. These techniques must never be applied to any system without explicit written permission from the owner.

---

## 👤 Author

**Bilal Ashfaq**

`Computer Science Student | Cybersecurity & Networking Enthusiast`

**Program:** Networkwalks Cybersecurity Internship (Batch B083)

LinkedIn: https://www.linkedin.com/in/bilal-siddiqui-61562a333/
