# OnlyOffice Document Server: Removal and Reinstallation Guide (Production)

**Author:** Abubakkar (System Admin)
**Date:** July 31, 2025
**Environment:** Production Linux (native OnlyOffice installation)

---

## 1. Purpose

Shopfloor.schertech.com’s DocVisu instance is impacted by a documented bug in older OnlyOffice versions that causes non-ZIP formats (such as DOCX) to be saved as ZIP archives. This behavior was reported and discussed in the OnlyOffice community forum (see [https://community.onlyoffice.com/t/onlyoffice-desktop-saves-docx-file-as-zip/3096/4](https://community.onlyoffice.com/t/onlyoffice-desktop-saves-docx-file-as-zip/3096/4)). While the discussion involved another user, the issue mirrors what was experienced here. A complete removal and clean reinstallation of the Document Server is required to restore proper file-format handling.

## 2. Prerequisites

* **Root** or **sudo** access
* VM snapshot or full backup available
* Logical backups of OnlyOffice data and PostgreSQL
* SSH connectivity to the server

## 3. Full Backup

1. **Database**

   ```bash
   sudo -u postgres pg_dump onlyoffice > ~/onlyoffice_db_backup_$(date +%F).sql
   ```
2. **Configuration & Data**

   ```bash
   sudo cp -a /etc/onlyoffice ~/backup/etc_onlyoffice_$(date +%F)
   sudo cp -a /var/www ~/backup/var_www_$(date +%F)
   ```
3. **Existing NGINX & OnlyOffice Config**

   ```bash
   sudo cp /etc/nginx/conf.d/ds.conf ~/backup/ds.conf_$(date +%F)
   sudo cp /etc/onlyoffice/documentserver/local.json ~/backup/local.json_$(date +%F)
   ```

## 4. Stop OnlyOffice Services

```bash
sudo systemctl stop ds-*
# Confirm all have stopped
ps aux | grep -E 'documentserver|ds-'
```

## 5. Purge OnlyOffice Packages

### 5.1 Enterprise Edition (Optional)

```bash
sudo apt purge onlyoffice-documentserver-ee
```

*If not installed, apt reports “not installed”—this is expected.*

### 5.2 Standard Edition

```bash
sudo apt purge onlyoffice-documentserver
```

#### 5.2.1 NGINX `ds.conf` Pre-Removal Error

* **Error:**

  ```text
  unlink: cannot unlink '/etc/nginx/conf.d/ds.conf': No such file or directory
  ```
* **Remediation:**

```bash
sudo mkdir -p /etc/nginx/conf.d
sudo touch /etc/nginx/conf.d/ds.conf
sudo dpkg --configure -a
sudo apt purge onlyoffice-documentserver
```

#### 5.2.2 Manual Cleanup (No `autoremove`)

Remove only OnlyOffice–specific directories; preserve other packages:

```bash
sudo rm -rf /var/www/onlyoffice
sudo rm -rf /etc/onlyoffice/
sudo rm -rf /var/lib/onlyoffice
sudo rm -rf /etc/nginx/includes/ds.conf
```

Re-run purge if needed:

```bash
sudo apt purge onlyoffice-documentserver
```

## 6. Verify Clean Removal

```bash
dpkg -l | grep onlyoffice   # Expect no entries
systemctl list-unit-files | grep ds-   # Expect none
```

## 7. Reinstallation Procedure

Below is the chosen installation workflow for a clean, production-ready setup.

### 7.1 Install Dependencies

#### 7.1.1 PostgreSQL

Before creating a fresh database, drop any existing OnlyOffice database and user to avoid conflicts:

```bash
sudo -i -u postgres psql -c "DROP DATABASE IF EXISTS onlyoffice;"
sudo -i -u postgres psql -c "DROP USER IF EXISTS onlyoffice;"
```

Proceed with installation and database creation: [If already exists **skip**.]

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl enable --now postgresql.service
```

Create user & database:

```bash
sudo -i -u postgres psql -c "CREATE USER onlyoffice WITH PASSWORD 'onlyoffice';"
sudo -i -u postgres psql -c "CREATE DATABASE onlyoffice OWNER onlyoffice;"
```

#### 7.1.2 RabbitMQ

```bash
sudo apt install rabbitmq-server
sudo systemctl enable --now rabbitmq-server.service
```

### 7.2 Configure Debconf Pre-selections

Use `debconf-set-selections` to define ports and database credentials before package install:

```bash
echo "onlyoffice-documentserver onlyoffice/ds-port select 8088" | sudo debconf-set-selections
echo "onlyoffice-documentserver onlyoffice/db-user string onlyoffice" | sudo debconf-set-selections
echo "onlyoffice-documentserver onlyoffice/db-pwd password onlyoffice" | sudo debconf-set-selections
```

### 7.3 Add Repository & GPG Key

```bash
# Import GPG key securely
gpg --no-default-keyring --keyring /tmp/onlyoffice.gpg \
    --import <(curl -fsSL https://download.onlyoffice.com/GPG-KEY-ONLYOFFICE)
sudo install -o root -g root -m 644 /tmp/onlyoffice.gpg \
    /usr/share/keyrings/onlyoffice.gpg

# Add repo entry
echo "deb [signed-by=/usr/share/keyrings/onlyoffice.gpg] \
    https://download.onlyoffice.com/repo/debian squeeze main" \
    | sudo tee /etc/apt/sources.list.d/onlyoffice.list
sudo apt update
```

### 7.4 Install OnlyOffice Document Server

```bash
sudo apt install onlyoffice-documentserver
```

*(Use ****`onlyoffice-documentserver-ee`**** if Enterprise Edition is required and licensed.)*

### 7.5 Post-Install Configuration

1. **Backup new configs**:

   ```bash
   sudo cp /etc/onlyoffice/documentserver/local.json \
       /etc/onlyoffice/documentserver/local.json.default
   ```
2. **Merge custom settings** (e.g., storage paths, SSL setups) back into `local.json`.
3. **Restore custom NGINX config**:

   ```bash
   sudo cp ~/backup/ds.conf_$(date +%F) /etc/nginx/conf.d/ds.conf
   ```
   make changes like (Secrets)
4. **Validate and reload NGINX**:

   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```

## 8. Post-Installation Verification

1. **Service Status**:

   ```bash
   sudo systemctl status ds-documentserver
   ```
2. **Port Listening**:

   ```bash
   sudo ss -tulwn | grep 8088
   ```
3. **Functional Test**: Open `http://<server-ip>:8088/` in browser and verify editor loads.

## 9. Documentation & Audit

* Record actual ports, paths, and any deviations in runbook.
* Notify stakeholders and update CMDB.

---
© 2025 Abubakkar | System Admin
