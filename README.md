# IBM Resilience (SOAR) – Enterprise SOP

---

## 1. Installation SOP

### 1.1 Pre-Requisites

| Requirement | Details |
|------------|--------|
| SOAR Version | 51.0.6.2.23 |
| Deployment Type | Virtual Appliance (OVA) |
| Files | `soar-51.0.6.2.23.run` / `ova` |
| Hardware | As per IBM sizing documentation |
| SSL | RootCA.pem, cert.pem, key.pem, Intermediate.pem |
| FQDN | `soar.<yourdomain>` |
| Ports | 443, 65000, 65001 |
| License | `License.txt` |

**Important:** Values inside `< >` must be replaced with environment-specific values before execution.

---

### 1.2 Installation Steps

#### a. Login via ESXi Console
Access the SOAR appliance console from ESXi and complete the guided bootstrap configuration (network, disk, OS readiness).

---

#### b. Configure Base URL
Defines the canonical access URL for SOAR services and UI.

```bash
sudo resutil configset -baseurl 'https://soar.<yourdoamin>'
```

---

#### c. Verify Base URL
Confirms the base URL is correctly applied.

```bash
sudo resutil configget -baseurl
```

---

#### d. Convert SSL to PKCS12 Format
Combines private key, certificate, and CA chain into a Java-compatible keystore.

```bash
sudo openssl pkcs12 -export -out cert.p12 -inkey key.pem -in cert.pem -certfile RootCA.pem
```

---

#### e. Import PKCS12 into SOAR Keystore
Imports SSL material into SOAR’s internal Java keystore.

```bash
sudo keytool -importkeystore -srckeystore cert.p12 -srcstoretype pkcs12 -srcalias 1 \
-destkeystore keystore -destalias co3 \
-deststorepass "$(sudo resutil keyvaultget -name keystore)" \
-destkeypass "$(sudo resutil keyvaultget -name keystore)"
```

---

#### f. Backup and Replace Active Keystore
Ensures rollback capability before overwriting the active keystore.

```bash
sudo cp /crypt/certs/keystore /crypt/certs/keystore.save
sudo cp keystore /crypt/certs/keystore
```

---

#### g. Configure Hostname and Timezone
Ensures system identity consistency and correct event timestamps.

```bash
sudo hostnamectl set-hostname soar.<yourdoamin>
sudo timedatectl set-timezone Asia/Kathmandu
```

---

#### h. Restart Core Services
Applies SSL, hostname, and configuration changes.

```bash
sudo systemctl restart resilient resilient-messaging
```

---

#### i. Create Master User and Organization
Creates the initial administrative user and default organization.

```bash
sudo resutil newuser -createorg -email "your_name@yourdomain" \
-first "<First_Name>" -last "<Last_Name>" -org "<Your_Organization>"
```

---

#### j. Retrieve QRadar SSL Certificate
Exports QRadar’s certificate for trusted SOAR communication.

```bash
openssl s_client -connect <qradar_domain:443> -tls1_2 -showcerts </dev/null 2>/dev/null | \
openssl x509 -outform PEM > qradar.pem
```

---

#### k. Copy Certificate to Custom Store
Places third-party certificates in SOAR’s trusted location.

```bash
sudo cp qradar.pem /crypt/certs/custcerts
```

---

#### l. Import QRadar Certificate
Adds QRadar certificate to the SOAR trust store.

```bash
sudo keytool -importcert -trustcacerts -file qradar.pem -alias <qradar_doamin> -keystore /crypt/certs/custcerts
```

---

#### m. Restart Services
Reloads certificates and integrations.

```bash
sudo systemctl restart resilience resilience-messaging
```

---

#### n. Import SOAR License
Activates SOAR features.

```bash
sudo license-import –file <path_to_license_file>
```

---

#### o. Verify License Status
Ensures license validity.

```bash
sudo resutil license
```

---

#### p. Access SOAR Web UI

```
https://soar.<yourdomain>
```

---

## 2. Post-Installation Validation SOP

### 2.1 Service Validation

```bash
systemctl status resilient
systemctl status resilient-messaging
```

---

### 2.2 Platform Validation Checklist

- HTTPS access without browser warnings
- License status shows **Valid**
- Correct hostname and timezone
- Organization and user visible
- No errors in `/var/log/resilient/`

---

## 3. QRadar–SOAR Integration SOP

### 3.1 Pre-Integration Checks

- QRadar API token created
- Network connectivity on port 443
- QRadar certificate imported

---

### 3.2 Integration Configuration

**SOAR UI → Administrator → Integrations → QRadar**

| Field | Value |
|------|------|
| QRadar Host | `https://<qradar_domain>` |
| Authentication | API Token |
| SSL Verification | Enabled |
| Polling | Enabled |

---

### 3.3 Integration Validation

- Connection test successful
- Offenses ingested
- Artifacts synchronized

---

## 4. Upgrade & Rollback SOP

### 4.1 Pre-Upgrade Checklist

- Snapshot VM
- Backup `/crypt/certs`
- Export workflows and integrations
- Verify license validity

---

### 4.2 Upgrade Execution

```bash
sudo ./soar-<new_version>.run
```

---

### 4.3 Rollback Procedure

1. Power off VM
2. Restore snapshot
3. Validate services and license

---

## 5. Security Hardening Baseline

- Disable root SSH login
- Enforce key-based authentication
- Restrict firewall ports
- Apply least-privilege RBAC
- Rotate API tokens quarterly
- Monitor audit logs

---

## 6. Automated Health-Check Script

```bash
#!/bin/bash
echo "SOAR Health Check"

systemctl is-active resilient resilient-messaging || echo "Service issue"
resutil license | grep -i valid || echo "License issue"
curl -k https://localhost | grep -i html || echo "UI issue"

echo "Health check completed"
```

---

## 7. One-Page Quick Reference

```text
Login → Base URL → SSL → Restart
Create Org → QRadar Cert → License → Validate
```

---

**End of SOP**

