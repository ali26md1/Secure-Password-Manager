# 🔐 Implementing Security into a Secure Password Manager

A **self-hosted Vaultwarden deployment** hardened and security-tested in a controlled **Kali Linux + Docker laboratory environment**.

The purpose of this project was to apply practical cybersecurity controls to a password-management application and **verify their effectiveness through direct technical testing**, rather than relying solely on documentation or vendor claims.

> ⚠️ **Status:** Laboratory proof of concept. **Not production-ready.**
> See [Limitations](#limitations) and [Recommendations](#recommendations) before considering any real-world deployment.

---

## 📌 Project Overview

Password managers handle highly sensitive information, making security controls such as encryption, authentication, access control, secure backups, and monitoring especially important.

In this project, I deployed **Vaultwarden**, an unofficial Bitwarden-compatible server written in Rust, and implemented a defined set of security controls.

The deployment was then assessed using tools such as:

* `curl`
* `openssl`
* `sqlite3`
* `sha256sum`
* Docker tooling

A total of **20 test cases** were defined to validate the implemented controls.

---

## 🧪 Lab Environment

| Component      | Version                       |
| -------------- | ----------------------------- |
| Host OS        | Kali GNU/Linux Rolling 2026.3 |
| Docker         | 28.5.2                        |
| Docker Compose | 2.40.3                        |
| Vaultwarden    | 1.37.2                        |
| Web Vault      | 2026.7.0                      |
| SQLite         | 3.46.1                        |
| OpenSSL        | 3.6.3                         |

---

## 🛡️ Security Controls Implemented

### 1. Transport Security

HTTPS/TLS was configured using Rocket's built-in TLS support with a self-signed certificate for the laboratory environment.

Additional security headers were configured/tested, including:

* HSTS
* Content-Security-Policy (CSP)
* X-Frame-Options
* Permissions-Policy

> The self-signed certificate is intentionally used for this controlled lab and should be replaced with a certificate issued by a trusted CA for production.

### 2. Key Derivation

The deployment uses:

* **Argon2id**
* Memory: **32 MB**
* Iterations: **6**
* Parallelism: **4**

### 3. Vault Encryption

Vault data was examined for plaintext storage.

Database fields consistently carried the Bitwarden `2.` CipherString prefix, corresponding to encrypted vault data.

The project documents the use of:

* AES-256-CBC
* HMAC-SHA256
* Client-side field-level encryption

### 4. Multi-Factor Authentication

TOTP-based two-step authentication was configured.

Testing verified that authentication attempts without the required TOTP token were rejected.

### 5. Access Control

An organization and collection structure was configured with a restricted **User** role.

Testing verified that the restricted member:

* Could access only its assigned collection
* Could not access the Admin Console

### 6. Encrypted Backups

Database backups were:

1. Created using SQLite's backup functionality
2. Integrity-checked
3. Encrypted using AES-256-CBC
4. Protected using PBKDF2
5. Restored and compared against the original

Backup encryption parameters:

```text
AES-256-CBC
PBKDF2
200,000 iterations
```

The decrypted database was verified to be **byte-identical** to the original backup and passed:

```sql
PRAGMA integrity_check;
```

The full backup also includes `rsa_key.pem`, which is required for organization and sharing functionality.

### 7. Logging & Monitoring

The deployment was monitored using:

* Rocket request/response logs
* Authentication-event logging
* Docker health checks
* Docker resource checks

---

# 🚀 Deployment

## Prerequisites

The laboratory environment requires:

* Kali Linux or another compatible Linux environment
* Docker
* Docker Compose
* OpenSSL
* SQLite3

## Docker Compose

The final deployment configuration was:

```yaml
services:
  vaultwarden:
    container_name: vaultwarden
    image: vaultwarden/server:latest
    restart: unless-stopped
    environment:
      DOMAIN: "https://127.0.0.1:8443"
      ROCKET_TLS: '{certs="/ssl/cert.pem",key="/ssl/key.pem"}'
      SIGNUPS_ALLOWED: "true"
    ports:
      - "8443:80"
    volumes:
      - ./vw-data:/data
      - ./tls:/ssl:ro
```

Start the deployment:

```bash
docker-compose up -d
```

The service can then be accessed at:

```text
https://127.0.0.1:8443
```

Because the laboratory uses a self-signed certificate, the browser will display a certificate warning. This is expected in the lab environment.

---

# 💾 Backup & Recovery

### Create and verify database backup

```bash
sqlite3 vw-data/db.sqlite3 ".backup 'Backups/vaultwarden_db_backup.sqlite3'"

sqlite3 Backups/vaultwarden_db_backup.sqlite3 \
"PRAGMA integrity_check;"
```

### Encrypt the backup

```bash
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 200000 \
-in Backups/vaultwarden_db_backup.sqlite3 \
-out Backups/vaultwarden_db_backup.sqlite3.enc
```

### Decrypt and verify

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 \
-in Backups/vaultwarden_db_backup.sqlite3.enc \
-out /tmp/vaultwarden_db_recovered.sqlite3
```

Compare the original and recovered databases:

```bash
sha256sum \
Backups/vaultwarden_db_backup.sqlite3 \
/tmp/vaultwarden_db_recovered.sqlite3
```

The recovery test produced a byte-identical database that successfully passed SQLite integrity verification.

---

# 🔎 Security Testing

A total of **20 test cases** were performed against the live deployment.

### Testing tools

| Tool        | Purpose                               |
| ----------- | ------------------------------------- |
| `curl`      | HTTP/HTTPS and authentication testing |
| `openssl`   | TLS and cryptographic testing         |
| `sqlite3`   | Database and integrity testing        |
| `sha256sum` | File/hash comparison                  |
| Docker      | Container health and resource checks  |

### Results

| Result                  |  Count |
| ----------------------- | -----: |
| ✅ Passed                |     17 |
| ⚠️ Inconclusive         |      1 |
| ⏳ Not tested            |      1 |
| **Total defined tests** | **20** |

> TC-02 and TC-16 passed with documented caveats.

### Testing highlights

The assessment verified that:

* Vault database fields consistently contained encrypted CipherString data
* Plaintext vault content was not observed at rest
* Invalid master-password authentication attempts were rejected
* Missing TOTP authentication was rejected
* Authentication failures were logged
* Restricted users could access only their assigned collection
* Restricted users had no Admin Console access
* Encrypted backups could be successfully restored
* Recovered databases were byte-identical to the original backups
* Recovered databases passed SQLite integrity verification

---

# ⚠️ Findings & Limitations

This project is intentionally documented as a **laboratory proof of concept** rather than a production deployment.

## Open Findings

### Administrator Token

`ADMIN_TOKEN` was observed being stored in plaintext, with Vaultwarden generating a recurring notice.

Further remediation is required.

### RSA Key Permissions

`rsa_key.pem` was observed with both `644` and `600` permissions at different points during the project.

The final permission state remains **unconfirmed**.

### User Registration

The final configuration contains:

```yaml
SIGNUPS_ALLOWED: "true"
```

This should be reviewed and disabled after intended accounts have been created.

### Rate Limiting

No rate-limiting or account-lockout mechanism was evidenced during this assessment.

---

## Not Tested

The following areas were outside the scope or constraints of this laboratory assessment:

* Mobile applications
* Desktop applications
* Browser-extension autofill
* Emergency Access
* SSO
* Duo
* Bitwarden Send
* Load/performance testing
* Full penetration testing
* Full disaster-recovery rebuild on a separate machine

---

# 🔧 Recommendations

Before considering a production deployment:

1. Convert `ADMIN_TOKEN` to an Argon2 PHC hash
2. Set and verify `rsa_key.pem` permissions to `600`
3. Set `SIGNUPS_ALLOWED=false` after creating intended accounts
4. Replace the self-signed certificate with a trusted CA certificate
5. Deploy behind an appropriately configured reverse proxy
6. Implement rate-limiting or fail2ban-style protection
7. Forward logs to a centralized logging/monitoring system
8. Configure alerting for relevant authentication/security events
9. Implement backup rotation
10. Maintain an off-host backup
11. Perform a complete disaster-recovery exercise

---

# 📁 Repository Contents

```text
Secure-Password-Manager/
│
├── Secure_Password_Manager_Project_-_Final_Report.pdf
├── docker-compose.yml
├── Backups/
├── vw-data/
├── tls/
└── ...
```

The repository includes the **full project report**, containing:

* Project methodology
* Security controls
* Threat model
* Test cases
* Technical evidence
* Findings
* Limitations
* Recommendations

---

# 🎯 Skills Demonstrated

Through this project, I gained practical experience with:

* 🔐 Authentication & MFA
* 🔑 Encryption & cryptography
* 🛡️ Security hardening
* 🌐 HTTPS/TLS
* 🐳 Docker & containerized deployment
* 🐧 Linux
* 🔎 Security testing
* 📊 Log analysis
* 💾 Secure backup & recovery
* 🔒 Access control
* 🧪 Technical verification
* 🚨 Security findings & remediation planning

---

# 📄 Project Report

The complete technical report is available in the repository:

**`Secure_Password_Manager_Project_-_Final_Report.pdf`**

It contains the detailed methodology, evidence, threat model, test cases, findings, and recommendations.

---

# 👨‍💻 Author

**Mohammed Ali**

IT Graduate | Aspiring Cybersecurity Professional

---

# ⚖️ Disclaimer

This project was developed and tested for **educational and laboratory purposes**.

The deployment described in this repository is **not production-ready**. The documented findings and limitations should be addressed before any real-world deployment.

---

## 📜 License

This project is intended for educational purposes.

If you would like to reuse or modify the project, consider adding an appropriate open-source license such as MIT.
