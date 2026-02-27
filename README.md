# Oracle 19c Backup Script (expdp + cron + rclone)

## Overview

Script สำหรับ Backup Oracle Database แบบ Full Export ด้วย expdp\
รองรับหลาย Database และอัปโหลดไป Google Drive ผ่าน rclone\
ทำงานอัตโนมัติด้วย cron

------------------------------------------------------------------------

## 1️⃣ Prerequisites

-   Oracle Database 19c ติดตั้งเรียบร้อย
-   User oracle ใช้งานได้
-   rclone config เรียบร้อย
-   Directory DATA_PUMP_DIR ชี้ไปที่ dpdump ของแต่ละ DB

------------------------------------------------------------------------

## 2️⃣ Enable OS Authentication

ทดสอบก่อน:

    sqlplus / as sysdba

ถ้าเข้าได้โดยไม่ถาม password = OK

ถ้าไม่ได้ ให้แก้ไฟล์:

    $ORACLE_HOME/network/admin/sqlnet.ora

เพิ่ม:

    SQLNET.AUTHENTICATION_SERVICES = (BEQ)

แล้ว restart database:

    sqlplus / as sysdba
    shutdown immediate;
    startup;

------------------------------------------------------------------------

## 3️⃣ ทดสอบ expdp

    unset TWO_TASK
    $ORACLE_HOME/bin/expdp "'/ as sysdba'" FULL=Y DIRECTORY=DATA_PUMP_DIR DUMPFILE=test.dmp LOGFILE=test.log

------------------------------------------------------------------------

## 4️⃣ Backup Script

ไฟล์: /home/oracle/backup_all_db.sh

    #!/bin/bash
    
    # ===== Oracle Environment =====
    export ORACLE_BASE=/u01/app/oracle
    export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
    export PATH=$ORACLE_HOME/bin:$PATH
    
    DATE=$(date +%Y%m%d)
    
    DB_LIST=(
    db1
    db2
    db3
    )
    
    for DB in "${DB_LIST[@]}"
    do
        echo "===== Backup $DB ====="

    export ORACLE_SID=$DB
    DUMP_DIR=/u01/app/oracle/admin/$DB/dpdump

	$ORACLE_HOME/bin/expdp \'sys/password as sysdba\' \
		FULL=Y \
		DIRECTORY=DATA_PUMP_DIR \
		DUMPFILE=${DB}_$DATE.dmp \
		LOGFILE=${DB}_$DATE.log

    # เช็คว่ามีไฟล์จริงก่อน upload
    if [ -f "$DUMP_DIR/${DB}_$DATE.dmp" ]; then
        rclone copy "$DUMP_DIR/${DB}_$DATE.dmp" gdrive:OracleBackup/$DB/
    else
        echo "Backup failed for $DB"
    fi

    # ลบไฟล์เก่าเกิน 7 วัน
    find "$DUMP_DIR" -name "*.dmp" -mtime +7 -delete

done

echo "===== Backup Completed ====="

ให้สิทธิ์ execute:

    chmod +x /home/oracle/backup_all_db.sh

------------------------------------------------------------------------

## 5️⃣ ตั้งค่า Cron

    crontab -e

เพิ่ม:

    05 02 * * * /home/oracle/backup_all_db.sh >> /home/oracle/backup_cron.log 2>&1

------------------------------------------------------------------------

## 6️⃣ ตรวจสอบ Log

    cat /home/oracle/backup_cron.log

------------------------------------------------------------------------

## Troubleshooting

### ถาม Password

-   ตรวจสอบ sqlplus / as sysdba
-   ตรวจสอบ SQLNET.AUTHENTICATION_SERVICES = (BEQ)
-   ใช้ unset TWO_TASK

### LRM-00108

เกิดจาก quoting ผิด\
ต้องใช้:

    expdp "'/ as sysdba'"

------------------------------------------------------------------------

## Result

-   Backup อัตโนมัติทุกวัน
-   ไม่ถาม password
-   ลบไฟล์เก่าเกิน 7 วัน
-   Upload ขึ้น Google Drive อัตโนมัติ
