# วิธีการตั้งค่า PostgreSQL High Availability (HA)
การที่ทำให้ PostgreSQL เป็นแบบ HA นั้นเพื่อให้ระบบ PostgeSQL ของเราให้มี Donwtime น้อยที่สุด ลดการ Failover, data redundancy ให้น้อยที่สุดโดยมีวิธีการตั้งต่าดังต่อไปนี้

## 1. Streaming Replication and Failover
การทำ Streaming Replication and Failover คือการตั้งค่าให้ Primary Server ทำการส่งค่าไปยัง Replica Server เพื่อให้ข้อมูลมีข้อมูลอัพเดทข้อมูลตรงกันระหว่าง Primary และ Replica server ให้ทำตามขั้นตอนต่อไปนี้

### 1.1 Primary Server Configuration
- เปิดการใช้งาน WAL (Write-Ahead Logging) and replication ตั้งค่าในไฟล์ `/etc/postgresql/[POSTGRESQL Version]/main/postgresql.conf`:
```bash
wal_level = replica
max_wal_senders = 5
wal_keep_segments = 64
```

- เพิ่ม replication user ในไฟล์ `/etc/postgresql/[POSTGRESQL Version]/main/pg_hba.conf`:
```bash
host replication replicator [192.168.x.x => IP เครื่องปลายทาง]/24 md5
```

### 1.2 Replica Server Configuration:
- ตั้งค่าให้เครื่อง Replica ทำการ Backup ข้อมูลจากเครื่อง Primary
```bash
pg_basebackup -h primary_host -D /var/lib/postgresql/[POSTGRESQL Version]/main -U replicator -v -P
```

- สร้างไฟล์ `recovery.conf` ใน data directory:
```bash
standby_mode = 'on'
primary_conninfo = 'host=primary_host port=5432 user=replicator password=yourpassword'
trigger_file = '/tmp/pg_failover_trigger'
```

### 1.3 Monitoring and Failover:
- ใช้เครื่องมือ repmgr หรือ Patroni เพื่อใช้ในการ monitor การ Fialover
- ตั้งค่าให้ repmgr สามารถจัดการ replication ได้
```bash
repmgr standby clone -f /etc/repmgr.conf
repmgr standby register -f /etc/repmgr.conf
```

## 2. Patroni and Etcd/Consul
- Patroni คือระบบที่ใช้ในการทำ PostgreSQL HA solution ใช้ในการตั้งค่าการกระจายของข้อมูล Etcd หรือ Consul เพื่อจัดการ PostgreSQL clusters ตามขั้นตอนต่อไปนี้
1. ติดตั้ง Dependencies
   - ติดตั้ง PostgreSQL, Patroni, and Etcd/Consul.
3. Configure Etcd/Consul:
   - Run Etcd/Consul cluster nodes พร้อมตั้งค่ราตามข้อ 1 ให้เรียบร้อย
5. Configure Patroni:
   - สร้างไฟล์ patroni.yml สำหรับการตั้งค่า:
```bash
scope: postgresql-ha
namespace: /service/
name: pg-node-1

etcd:
  host: 127.0.0.1:2379

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.0.1:5432
  data_dir: /var/lib/postgresql/[POSTGRESQL Version]/main
  bin_dir: /usr/lib/postgresql/[POSTGRESQL Version]/bin
  authentication:
    superuser:
      username: postgres
      password: [POSTGRES Password]
    replication:
      username: replicator
      password: [REPLICATOR Password]
  parameters:
    max_connections: 100
    shared_buffers: 256MB

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.0.1:8008
```

 - Start Patroni
```bash
patroni /etc/patroni.yml
```

## 3. pgpool-II
- Pgpool-II คือ middleware ที่ทำงานระหว่าง PostgreSQL servers และ clients โดยจะคอยจัดการ Connection pooling, load balancing, และ automated failover มีการตั้งค่าดังต่อไปนี้

### 3.1 ติดตั้ง Pgpool-II:
  - ติดตั้ง Pgpool-II บน Server ที่แยกออกมาจาก Server ของ PostgreSQL หรือจะติดตั้งในเครื่องเดียวกันก็ได้
### 3.2	ตั้งค่า Pgpool-II:
	- แก้ไขไฟล์ /etc/postgresql/[POSTGRESQL Version]/main/pgpool.conf:
```bash
listen_addresses = '*'
backend_hostname0 = '192.168.0.1'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/var/lib/postgresql/12/main'
backend_flag0 = 'ALLOW_TO_FAILOVER'

backend_hostname1 = '192.168.0.2'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/var/lib/postgresql/12/main'
backend_flag1 = 'ALLOW_TO_FAILOVER'

enable_pool_hba = on
pool_passwd = 'pool_passwd'

sr_check_user = 'replicator'
sr_check_password = 'replicatorpassword'
```

### 3.3	ตั้งค่า Failover Script:
	-	ตั้งค่า failover script ในไฟล์ pgpool.conf:
 bash```
 failover_command = '/etc/pgpool-II/failover.sh %d %P %H'
 ```

- สร้างไฟล์ failover script (/etc/pgpool-II/failover.sh):
```bash
#!/bin/bash
failed_node=$1
new_primary=$3
pcp_recovery_node -h localhost -U pgpool -p 9898 $failed_node
pcp_promote_node -h localhost -U pgpool -p 9898 $new_primary
```

เป็นอันเสร็จสิ้น
