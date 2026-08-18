# Information Disclosure – Reconnaissance & Testing Workflow

Information Disclosure occurs when an application unintentionally exposes sensitive information to unauthorized users.

Examples include:

* API keys and credentials
* Internal IP addresses
* Source code
* Database information
* Stack traces
* Server paths
* Software versions
* Internal API endpoints
* Other users' personal information
* Configuration files and backup files

Finding Information Disclosure requires a structured reconnaissance and testing process. The following workflow moves from **passive reconnaissance** to **active application testing**.

> **⚠️ Authorization Notice:** Only perform active scanning, fuzzing, parameter manipulation, or vulnerability testing against systems you own or have explicit permission to test, such as authorized labs or bug bounty targets.

---

## 1. Passive Reconnaissance

### Goal

Collect publicly available information without directly interacting with the target infrastructure.

### 1.1 Public Code Repositories

Search repositories on platforms such as GitHub and GitLab for information accidentally committed by developers.

### What to Search For

* Domain names
* API keys
* Passwords
* Tokens
* Database credentials
* Internal IP addresses
* Configuration files
* Environment variables

### Example Search Queries

```text
site:github.com company.com
site:github.com "company.com" password
site:github.com "company.com" api_key
site:github.com "company.com" secret
```

### Possible Findings

```text
AWS credentials
Database passwords
API tokens
Internal hostnames
Private IP addresses
Source code
Configuration files
```

---

## 2. Wayback Machine

### Goal

Identify information that was publicly accessible in older versions of the website.

Use the Internet Archive's Wayback Machine to inspect historical snapshots.

### Look For

```text
/admin/
/backup.zip
/config.json
/.env
/api/
/debug/
```

An old version of a website may contain information that has since been removed from the live application.

> Historical snapshots can reveal previously exposed content, but availability in an archive does not necessarily mean the information is currently accessible.

---

## 3. Search Engine Reconnaissance

Search engines can index files and directories that were unintentionally exposed.

### Example Queries

```text
site:example.com intitle:"index of"
site:example.com filetype:log
site:example.com filetype:sql
site:example.com filetype:env
site:example.com filetype:bak
```

### Look For

* Log files
* Backup files
* Database dumps
* Configuration files
* Directory listings
* Documentation
* Internal development pages

---

# 4. Subdomain Enumeration

### Goal

Discover additional applications and environments belonging to the target.

Common environments include:

```text
www.example.com
dev.example.com
test.example.com
staging.example.com
api.example.com
admin.example.com
```

### Tools

* Subfinder
* Amass
* DNS enumeration tools

### Example

```bash
subfinder -d example.com
```

### Information Disclosure Checks

For authorized targets, inspect discovered applications for:

* Debug pages
* Developer comments
* Internal API endpoints
* Development information
* Software versions
* Configuration details

---

# 5. Port and Service Discovery

### Goal

Identify exposed services that may reveal information about the infrastructure.

For authorized systems, Nmap can be used to identify open ports and service versions.

```bash
nmap -sV -p- example.com
```

### Interesting Services

```text
8080  → Alternative web application
6379  → Redis
27017 → MongoDB
9200  → Elasticsearch
```

An exposed service does **not automatically mean a vulnerability**. Verify whether it is intentionally exposed and whether authentication and access controls are properly configured.

---

# 6. Directory and File Enumeration

### Goal

Discover files and directories that are not linked from the normal application interface.

Tools commonly used:

* Gobuster
* Dirb
* Feroxbuster

### Example

```bash
gobuster dir \
-u https://example.com \
-w /path/to/common.txt \
-x php,json,yaml,bak,sql
```

### Files and Paths to Check

```text
/.git/
/.env
/swagger.json
/api-docs
/debug/
/phpinfo.php
/backup/
/config/
```

### Potential Information Disclosure

| Resource       | Possible Information                   |
| -------------- | -------------------------------------- |
| `.git`         | Source code and commit history         |
| `.env`         | Configuration and secrets              |
| `swagger.json` | API structure and endpoints            |
| `phpinfo.php`  | Server and PHP configuration           |
| `.log`         | Application and system information     |
| `.bak`         | Previous configuration or source files |
| Database dump  | Database information                   |

---

# 7. JavaScript and Source Code Inspection

Modern web applications send large amounts of JavaScript to the browser.

### Steps

1. Open browser Developer Tools.
2. Go to **Sources** or **Network**.
3. Reload the application.
4. Identify JavaScript files.
5. Search the source code for interesting strings.

### Useful Search Terms

```text
http://
https://
api
token
key
secret
admin
internal
localhost
192.168.
10.
debug
```

### Possible Findings

```text
Internal API endpoints
Internal hostnames
Development URLs
Debug endpoints
Hardcoded configuration
Client-side secrets
```

> Anything delivered to the browser should be considered accessible to the client. True secrets should never be embedded in frontend JavaScript.

---

# 8. HTTP Response and Header Inspection

Inspect HTTP responses using browser Developer Tools or Burp Suite.

### Check

```text
Status Code
Response Headers
Response Body
Cookies
CORS Headers
Server Information
Technology Information
Error Messages
```

### Example

```http
HTTP/1.1 200 OK
Server: Apache/2.4.x
X-Powered-By: Express
```

Headers such as `Server` and `X-Powered-By` may reveal technology information.

> Version disclosure can assist attackers with fingerprinting, but a disclosed version alone does not prove that the application is vulnerable.

---

# 9. Parameter Testing and Access-Control Checks

### Goal

Determine whether modifying request parameters exposes information belonging to another user or resource.

Use an authorized test account and an application you are permitted to assess.

### Example

Original request:

```http
GET /invoice/1001
```

Test:

```http
GET /invoice/1002
```

If the second resource belongs to another user and the application returns it without proper authorization, this may indicate an **Insecure Direct Object Reference (IDOR)** or broader **Broken Access Control** issue.

### Possible Information Disclosure

```text
Username
Email address
Phone number
Address
Invoice information
Account information
Private documents
```

This is more than simple information disclosure because the root cause may be an **authorization/access-control failure**.

---

# 10. Error Message Testing

### Goal

Determine whether invalid input causes the application to reveal internal implementation details.

For authorized testing, try unexpected or malformed input in application fields and API parameters.

Examples:

```text
'
"
\
%
Invalid data
Unexpected parameter values
```

### Look For

```text
Stack traces
File paths
Database errors
SQL queries
Table names
Column names
Programming language
Framework information
Internal hostnames
```

### Example of a Potential Leak

```text
File "/var/www/app/database.py", line 42
```

This may reveal:

* Operating system paths
* Application language
* Source-code locations
* Framework information
* Internal implementation details

Production applications should generally return controlled error messages instead of detailed stack traces.

---

# 11. Information Disclosure Testing Checklist

* [ ] Search public GitHub/GitLab repositories
* [ ] Search for exposed credentials and secrets
* [ ] Check historical website snapshots
* [ ] Review `robots.txt`
* [ ] Review `sitemap.xml`
* [ ] Perform authorized subdomain enumeration
* [ ] Identify exposed services and versions
* [ ] Enumerate directories and files
* [ ] Check for `.git`, `.env`, `.bak`, `.log`, and configuration files
* [ ] Check API documentation such as Swagger/OpenAPI
* [ ] Inspect JavaScript files
* [ ] Search JavaScript for internal endpoints and configuration
* [ ] Inspect HTTP response headers
* [ ] Review HTTP response bodies
* [ ] Test authorization around object/resource identifiers
* [ ] Test error handling with malformed input
* [ ] Check for stack traces and internal paths
* [ ] Document every confirmed disclosure with evidence

---

# 12. Recommended Testing Flow

```text
                 Information Disclosure Testing
                            │
                            ▼
                   Passive Reconnaissance
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          GitHub         Wayback       Search Engines
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                  Subdomain Enumeration
                            │
                            ▼
                   Service Enumeration
                            │
                            ▼
                 Directory/File Discovery
                            │
                            ▼
                JavaScript Inspection
                            │
                            ▼
                 HTTP Response Analysis
                            │
                            ▼
               Parameter/Access Testing
                            │
                            ▼
                    Error Testing
                            │
                            ▼
              Confirm → Document → Report
```

---

# 13. Important Distinction

Not every piece of technical information is automatically a vulnerability.

For example:

```text
Server: nginx
X-Powered-By: Express
```

is information disclosure/fingerprinting, but its severity may be low.

On the other hand:

```text
Database password
Private API token
Another user's personal information
Private source code
```

can represent a much more serious security issue.

Always consider:

```text
What information was exposed?
Who can access it?
Was the information supposed to be public?
What could an attacker do with it?
What is the impact?
```

---

## Key Takeaway

Information Disclosure testing is not about finding one specific endpoint. It is about systematically examining **everything the application exposes**.

A good testing mindset is:

```text
Recon
  ↓
Discover
  ↓
Inspect
  ↓
Manipulate (only when authorized)
  ↓
Confirm
  ↓
Assess Impact
  ↓
Document
```

The most important habit is to inspect both **requests and responses**, because sensitive information may be hidden in headers, JSON responses, JavaScript, error messages, source code, or forgotten files even when it is not visible in the application's normal interface.
