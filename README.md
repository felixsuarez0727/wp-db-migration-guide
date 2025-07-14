# Migration Guide: WordPress from Azure Flexible MySQL to VM with MySQL

## Process Overview
This guide will migrate a WordPress database from Azure Flexible MySQL Server to a VM with MySQL Server installed, while keeping the App Service running.

## Prerequisites Completed ✅
- Azure VM with SSH access
- MySQL Server installed and configured on the VM
- Successful connection from VM to Azure Flexible Server

---

## PHASE 1: BACKUP AND SAFETY MEASURES (CRITICAL)

### STEP 1: Complete Backup Before Migration

#### 1.1 Backup WordPress Files from App Service
In Azure Portal → App Service → Advanced Tools → Go → Debug console → CMD:
```bash
cd /home/site/wwwroot
# Create backup of all WordPress files
tar -czf wordpress_files_backup_$(date +%Y%m%d_%H%M%S).tar.gz .
# Download this file to your local machine
```

#### 1.2 Backup Current wp-config.php
```bash
cd /home/site/wwwroot
cp wp-config.php wp-config.php.pre-migration-backup
ls -la wp-config*
```

#### 1.3 Document Current Environment Variables
In Azure Portal → App Service → Environment variables:
**IMPORTANT:** Take screenshots or document ALL current values:
- `DATABASE_HOST`
- `DATABASE_NAME`
- `DATABASE_USERNAME` 
- `ENABLE_MYSQL_MANAGED_IDENTITY`
- Any other DATABASE or MYSQL related variables

#### 1.4 Create Complete Database Backup from Azure Flexible Server
```bash
# From your VM, create a timestamped backup
mysqldump -h [YOUR_FLEXIBLE_SERVER].mysql.database.azure.com \
  -u [YOUR_USERNAME] -p \
  --default-character-set=utf8mb4 \
  --skip-lock-tables \
  --single-transaction \
  --routines --triggers --events \
  --all-databases > full_backup_$(date +%Y%m%d_%H%M%S).sql

# Also create WordPress-specific backup
mysqldump -h [YOUR_FLEXIBLE_SERVER].mysql.database.azure.com \
  -u [YOUR_USERNAME] -p \
  --default-character-set=utf8mb4 \
  --skip-lock-tables \
  --single-transaction \
  --routines --triggers --events \
  [YOUR_DATABASE_NAME] > wordpress_backup_$(date +%Y%m%d_%H%M%S).sql
```

#### 1.5 Test Website Functionality BEFORE Migration
1. **Access the website** and confirm it loads
2. **Login to WordPress admin**
3. **Create a test post** titled "PRE-MIGRATION TEST"
4. **Verify the post appears** on the website
5. **Document the current post count:**
   ```bash
   mysql -h [YOUR_FLEXIBLE_SERVER].mysql.database.azure.com -u [YOUR_USERNAME] -p
   USE [YOUR_DATABASE_NAME];
   SELECT COUNT(*) FROM wp_posts;
   ```

#### 1.6 Create Rollback Documentation
Document the exact steps to return to current state:
```bash
# Current configuration to restore if needed:
DATABASE_HOST=[CURRENT_VALUE]
DATABASE_NAME=[CURRENT_VALUE]
DATABASE_USERNAME=[CURRENT_VALUE]
ENABLE_MYSQL_MANAGED_IDENTITY=[CURRENT_VALUE]
```

**⚠️ CRITICAL:** Do not proceed to Phase 2 until ALL backups are complete and verified.

---

## PHASE 2: PREPARATION AND EXPORT

### STEP 2: Verify Connection to Source Database

#### 2.1 Connect to Azure Flexible Server from VM
```bash
mysql -h [YOUR_FLEXIBLE_SERVER].mysql.database.azure.com -u [YOUR_USERNAME] -p
```

#### 2.2 Verify WordPress database
```sql
SHOW DATABASES;
USE [YOUR_DATABASE_NAME];
SHOW TABLES;
SELECT COUNT(*) FROM wp_posts;
```

**Expected result:** You should see WordPress tables and the original post count.

---

### STEP 3: Export the Database

#### 3.1 Create backup from VM
```bash
mysqldump -h [YOUR_FLEXIBLE_SERVER].mysql.database.azure.com \
  -u [YOUR_USERNAME] -p \
  --default-character-set=utf8mb4 \
  --skip-lock-tables \
  --single-transaction \
  --routines --triggers --events \
  [YOUR_DATABASE_NAME] > wordpress_dump.sql
```

#### 3.2 Verify file was created correctly
```bash
ls -lh wordpress_dump.sql
head -20 wordpress_dump.sql
```

**Expected result:** File should have considerable size and show SQL comments at the beginning.

---

## PHASE 3: VM PREPARATION

### STEP 4: Prepare Destination Database

#### 4.1 Connect to local MySQL on VM
```bash
sudo mysql -u root -p
```

#### 4.2 Create WordPress database
```sql
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

#### 4.3 Create user for remote connection (from App Service)
```sql
CREATE USER 'wp_remote'@'%' IDENTIFIED BY 'YOUR_SUPER_SECURE_PASSWORD';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_remote'@'%';
FLUSH PRIVILEGES;
```

#### 4.4 Verify creation
```sql
SHOW DATABASES;
SHOW GRANTS FOR 'wp_remote'@'%';
EXIT;
```

---

### STEP 5: Import the Database

#### 5.1 Import SQL file
```bash
# Use sudo if necessary
sudo mysql wordpress < wordpress_dump.sql
```

**Note:** This process may take several minutes depending on database size.

---

### STEP 6: Verify Migration

#### 6.1 Connect to local database
```bash
mysql -u wp_remote -p
# Or if it doesn't work: sudo mysql -u root -p
```

#### 6.2 Verify import
```sql
USE wordpress;
SHOW TABLES;
SELECT COUNT(*) FROM wp_posts;
SELECT COUNT(*) FROM wp_users;
SELECT post_title FROM wp_posts LIMIT 5;
```

**Checkpoint:** Data must match exactly with the original database.

---

## PHASE 4: NETWORK AND CONNECTIVITY CONFIGURATION

### STEP 7: Configure MySQL for External Connections

#### 7.1 Edit MySQL configuration
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Change:
```
bind-address = 127.0.0.1
```
To:
```
bind-address = 0.0.0.0
```

#### 7.2 Restart MySQL
```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

#### 7.3 Verify it's listening on all interfaces
```bash
sudo ss -tlnp | grep 3306
```

**Expected result:** Should show `0.0.0.0:3306`

---

### STEP 8: Configure Azure Firewall

#### 8.1 Configure Network Security Group
In Azure Portal:
1. Go to your VM → **Networking** → **Network Security Group**
2. **Add inbound port rule**:
   - **Source**: `Service Tag`
   - **Source service tag**: `AzureCloud`
   - **Source port ranges**: `*`
   - **Destination**: `Any`
   - **Destination port ranges**: `3306`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `100`
   - **Name**: `Allow-MySQL-AzureCloud`

#### 8.2 Get VM public IP
In Azure Portal → VM → Overview, note the **public IP**.

---

### STEP 9: Test Connectivity

#### 9.1 Test remote connection from VM
```bash
mysql -h [YOUR_VM_PUBLIC_IP] -u wp_remote -p wordpress -e "SELECT COUNT(*) FROM wp_posts;"
```

**Expected result:** Should connect and show post count.

---

## PHASE 5: APP SERVICE RECONFIGURATION

### STEP 10: Configure App Service for New Database

#### 10.1 Disable Managed Identity
In Azure Portal → App Service → Environment variables:

**Change:**
- `ENABLE_MYSQL_MANAGED_IDENTITY`: from `true` to **`false`**

#### 10.2 Configure new connection variables
**Change these variables:**
- `DATABASE_HOST`: `[YOUR_VM_PUBLIC_IP]`
- `DATABASE_NAME`: `wordpress`
- `DATABASE_USERNAME`: `wp_remote`

**Add new variable:**
- `DATABASE_PASSWORD`: `YOUR_SUPER_SECURE_PASSWORD`

#### 10.3 Save changes
App Service will restart automatically.

---

## PHASE 6: VERIFICATION AND TESTING

### STEP 11: Test Migration

#### 11.1 Wait for App Service restart (2-3 minutes)

#### 11.2 Verify website
Access your WordPress site and confirm it loads correctly.

#### 11.3 Test complete functionality
1. **Access WordPress admin** (`/wp-admin/`)
2. **Create a new post** with title: "MIGRATION TEST - [date/time]"
3. **Publish the post**

#### 11.4 Verify in database
```bash
# On your VM
mysql -u wp_remote -p wordpress

# In MySQL
SELECT post_title, post_date FROM wp_posts ORDER BY post_date DESC LIMIT 5;
```

**Expected result:** You should see the "MIGRATION TEST" post in your VM database.

---

## PHASE 7: FINAL VALIDATION

### STEP 12: Database Comparison

#### 12.1 Check database on VM (new)
```sql
-- On VM
SELECT 'VM - New DB' as source, COUNT(*) as posts FROM wp_posts;
SELECT post_title, post_date FROM wp_posts ORDER BY post_date DESC LIMIT 3;
```

#### 12.2 Check original database (Azure)
```bash
# Connect to original Flexible Server
mysql -h [YOUR_FLEXIBLE_SERVER].mysql.database.azure.com -u [YOUR_USERNAME] -p

# Verify
USE [YOUR_ORIGINAL_DATABASE];
SELECT 'Azure - Original DB' as source, COUNT(*) as posts FROM wp_posts;
SELECT post_title, post_date FROM wp_posts ORDER BY post_date DESC LIMIT 3;
```

**Successful validation:** 
- ✅ VM should have MORE posts (includes test)
- ✅ Azure should have FEWER posts (original data without test)

---

## COMMON TROUBLESHOOTING

### Error: "Error establishing a database connection"
**Cause:** Connectivity or authentication issue
**Solution:**
```bash
# Check environment variables in App Service
echo $DATABASE_HOST
echo $DATABASE_USERNAME
echo $DATABASE_PASSWORD

# Test connection from App Service to VM
curl -v telnet://[VM_IP]:3306 --connect-timeout 10
```

### Error: "Access denied for user"
**Cause:** Incorrect user or permissions
**Solution:**
```sql
-- In VM MySQL
SHOW GRANTS FOR 'wp_remote'@'%';
-- If doesn't exist, recreate:
CREATE USER 'wp_remote'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_remote'@'%';
FLUSH PRIVILEGES;
```

### Error: "Can't connect to MySQL server"
**Cause:** Firewall or network configuration
**Solution:**
1. Check Network Security Group in Azure
2. Confirm MySQL listens on `0.0.0.0:3306`
3. Verify VM public IP

---

## ROLLBACK COMMANDS (If something goes wrong)

### Complete Rollback to Pre-Migration State

#### Immediate Rollback (If migration fails)
```bash
# 1. Revert App Service Environment Variables
# In Azure Portal → App Service → Environment variables:
DATABASE_HOST=[ORIGINAL_FLEXIBLE_SERVER_VALUE]
DATABASE_NAME=[ORIGINAL_DATABASE_NAME]  
DATABASE_USERNAME=[ORIGINAL_USERNAME_VALUE]
ENABLE_MYSQL_MANAGED_IDENTITY=[ORIGINAL_VALUE - usually 'true']
# Remove DATABASE_PASSWORD if it was added
```

#### Full Restore from Backup (If data corruption occurs)
```bash
# 1. Restore WordPress files (if needed)
# Upload wordpress_files_backup_XXXXXXX.tar.gz to App Service
cd /home/site/wwwroot
tar -xzf wordpress_files_backup_XXXXXXX.tar.gz

# 2. Restore wp-config.php
cp wp-config.php.pre-migration-backup wp-config.php

# 3. Restore database (if Azure Flexible Server was damaged)
mysql -h [FLEXIBLE_SERVER].mysql.database.azure.com -u [USERNAME] -p [DATABASE_NAME] < wordpress_backup_XXXXXXX.sql
```

### Verify rollback worked
1. Restart App Service
2. Verify site loads with original functionality
3. Confirm "PRE-MIGRATION TEST" post exists
4. Confirm new migration test posts do NOT appear

---

## IMPORTANT NOTES

1. **Mandatory backup:** Keep backups of both databases during migration

2. **Minimal downtime:** Migration can be done with 5-10 minutes maximum downtime

3. **Managed Identity:** If using Managed Identity, it must be disabled to use password authentication

4. **Firewall:** Critical to configure Network Security Group correctly to allow connections from App Service

5. **Validation:** Always perform write tests to confirm WordPress uses the new database

6. **Cleanup:** Once migration is confirmed, you can delete Azure Flexible Server to save costs

---

## FINAL RESULT

Upon successful completion of this migration:

- ✅ **WordPress App Service** continues working normally
- ✅ **Database** is now on your VM with MySQL
- ✅ **Reduced costs** by eliminating Flexible Server
- ✅ **Full control** over database on your VM
- ✅ **Data preserved** and complete functionality

**Estimated duration:** 30-60 minutes depending on database size.

---

## SUCCESS CRITERIA

**Migration is successful when:**
1. WordPress website loads and functions normally
2. Admin access works with existing credentials
3. New posts/content save to VM database (not Azure)
4. Database comparison shows VM has new content while Azure remains unchanged
5. All WordPress functionality (posts, users, plugins, themes) works correctly

**Post-Migration Benefits:**
- **Cost savings:** Eliminate Azure Flexible Server charges
- **Performance control:** Direct management of MySQL configuration
- **Backup flexibility:** Full control over backup strategies
- **Scaling options:** Ability to scale VM resources as needed
