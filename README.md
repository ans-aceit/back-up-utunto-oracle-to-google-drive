# Ubuntu + Oracle 19c + Google Drive Backup Guide (Complete)

## Overview

คู่มือฉบับสมบูรณ์สำหรับ:
1-3 หากมีแล้วไม่ต้องทำ
1.  ติดตั้ง Ubuntu Server
2.  ติดตั้ง Oracle Database 19c
3.  ตั้งค่า OS Authentication
4.  ตั้งค่า rclone เชื่อมต่อ Google Drive
5.  สร้าง Backup Script (expdp)
6.  ตั้งค่า Cron ให้ Backup อัตโนมัติ

------------------------------------------------------------------------

# 1️⃣ ติดตั้ง Ubuntu Server

อัปเดตระบบ:

    sudo apt update
    sudo apt upgrade -y

	dnf update
	dnf upgrade -y
	

ติดตั้ง package ที่จำเป็น:

    sudo apt install -y unzip wget curl net-tools vim

------------------------------------------------------------------------

# 2️⃣ สร้าง User oracle

    sudo groupadd oinstall
    sudo groupadd dba
    sudo useradd -m -g oinstall -G dba oracle
    sudo passwd oracle

------------------------------------------------------------------------

# 3️⃣ ติดตั้ง Oracle 19c

ติดตั้ง dependencies:

    sudo apt install -y libaio1 libaio-dev

สร้าง directory:

    sudo mkdir -p /u01/app/oracle
    sudo chown -R oracle:oinstall /u01
    sudo chmod -R 775 /u01

สลับเป็น user oracle:

    su - oracle

ตั้งค่า environment (.bash_profile):

    export ORACLE_BASE=/u01/app/oracle
    export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
    export ORACLE_SID=orcl
    export PATH=$ORACLE_HOME/bin:$PATH

ติดตั้ง Oracle ตามขั้นตอน installer ของ 19c

------------------------------------------------------------------------

# 4️⃣ เปิด OS Authentication

แก้ไฟล์:

    $ORACLE_HOME/network/admin/sqlnet.ora

เพิ่ม:

    SQLNET.AUTHENTICATION_SERVICES = (BEQ)

Restart Database:

    sqlplus / as sysdba
    shutdown immediate;
    startup;

ทดสอบ:

    sqlplus / as sysdba

ต้องไม่ถาม password

------------------------------------------------------------------------

# 5️⃣ ติดตั้ง rclone (root)

    curl https://rclone.org/install.sh | sudo bash

ตั้งค่า: (oracle)

    rclone config

เลือก:

    n) New remote
    name: gdrive
    storage: drive

ทำตามขั้นตอน authorize ให้เสร็จ

ทดสอบ:

    rclone ls gdrive:

------------------------------------------------------------------------


# 7️⃣ สร้าง Backup Script

ไฟล์:

    nano /home/oracle/backup_all_db.sh

เนื้อหา:

	#!/bin/bash

	# ===== Oracle Environment =====
	export ORACLE_BASE=/u01/app/oracle
	export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
	export PATH=$ORACLE_HOME/bin:$PATH

	DATE=$(date +%Y%m%d)

	DB_LIST=(
	cesdbads
	cesdbbds
	cesdbtsc
	cesdbacc
	)

	for DB in "${DB_LIST[@]}"
	do
		echo "===== Backup $DB ====="

		export ORACLE_SID=$DB
		DUMP_DIR=/u01/app/oracle/admin/$DB/dpdump

			$ORACLE_HOME/bin/expdp \'sys/<Password> as sysdba\' \
					FULL=Y \
					DIRECTORY=DATA_PUMP_DIR \
					DUMPFILE=${DB}_$DATE.dmp \
					LOGFILE=${DB}_$DATE.log

		# เช็คว่ามีไฟล์จริงก่อน upload
		if [ -f "$DUMP_DIR/${DB}_$DATE.dmp" ]; then
			rclone copy "$DUMP_DIR/${DB}_$DATE.dmp" gdrive:OracleBackup/Database<DBIP>/$DB/
		else
			echo "Backup failed for $DB"
		fi

		# ลบไฟล์เก่าเกิน 7 วัน
		find "$DUMP_DIR" -name "*.dmp" -mtime +7 -delete

	done

	echo "===== Backup Completed ====="


ให้สิทธิ์:

    chmod +x /home/oracle/backup_all_db.sh

------------------------------------------------------------------------

# 8️⃣ ทดสอบรัน

    /home/oracle/backup_all_db.sh

------------------------------------------------------------------------

# 9️⃣ ตั้งค่า Cron

    crontab -e

เพิ่ม:

    05 02 * * * /home/oracle/backup_all_db.sh >> /home/oracle/backup_cron.log 2>&1

------------------------------------------------------------------------

# 🔟 ตรวจสอบ Log

    cat /home/oracle/backup_cron.log

------------------------------------------------------------------------

# ระบบ Backup ที่ได้

-   Backup Full Database ทุกวัน
-   ไม่ถาม password
-   ลบไฟล์เก่าเกิน 7 วัน
-   Upload ขึ้น Google Drive อัตโนมัติ
-   ทำงานผ่าน cron 100%

------------------------------------------------------------------------

# แนะนำ Production เพิ่มเติม

-   ใช้ COMPRESSION=ALL
-   ใช้ PARALLEL=4
-   ตั้ง retention บน Google Drive
-   ตั้ง alert mail เมื่อ backup fail
