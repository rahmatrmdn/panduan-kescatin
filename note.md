# Instalasi Kafka KRaft (Production Ready) — Ubuntu 24 LTS

> **Referensi config:** project `server.properties` & `zookeeper.properties`
> **Mode:** KRaft (tanpa ZooKeeper)
> **Cluster:** 3 node combined (controller + broker)
> **Kafka Version:** 3.9.2 (latest stable, full KRaft support)

## Arsitektur Cluster

| Node | IP | node.id | Roles |
|------|------|---------|-------|
| kafka-1 | `10.0.45.98` | 1 | controller, broker |
| kafka-2 | `10.0.45.99` | 2 | controller, broker |
| kafka-3 | `10.0.45.100` | 3 | controller, broker |

> [!NOTE]
> Config asli project menggunakan 3 node ZooKeeper + 3 broker di IP yang sama. Pada KRaft kita gabungkan role **controller** dan **broker** di setiap node (combined mode), sehingga tidak perlu ZooKeeper lagi.

---

## STEP 0 — Persiapan Semua Node

> [!IMPORTANT]
> Jalankan **semua step (0–7)** di **setiap 3 node** kecuali ada catatan khusus.

### 0.1 Update Sistem

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget curl net-tools tar openjdk-21-jdk-headless
```

### 0.2 Verifikasi Java

```bash
java -version
# Output: openjdk version "21.x.x"
```

### 0.3 Set JAVA_HOME

```bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' | sudo tee /etc/profile.d/java.sh
source /etc/profile.d/java.sh
```

### 0.4 Buat User Kafka (dedicated, non-root)

```bash
sudo useradd -r -m -s /usr/sbin/nologin kafka
```

---

## STEP 1 — Download & Install Kafka Binary

```bash
cd /opt
sudo wget https://downloads.apache.org/kafka/3.9.2/kafka_2.13-3.9.2.tgz
sudo tar -xzf kafka_2.13-3.9.2.tgz
sudo mv kafka_2.13-3.9.2 kafka
sudo rm kafka_2.13-3.9.2.tgz
```

### 1.1 Buat Direktori Data

```bash
# Direktori data broker (sesuai config asli: /data/kafka-logs)
sudo mkdir -p /data/kafka-logs

# Direktori metadata KRaft (pengganti ZooKeeper data)
sudo mkdir -p /data/kraft-metadata

# Set ownership
sudo chown -R kafka:kafka /data/kafka-logs
sudo chown -R kafka:kafka /data/kraft-metadata
sudo chown -R kafka:kafka /opt/kafka
```

---

## STEP 2 — Konfigurasi KRaft (`server.properties`)

> [!IMPORTANT]
> File ini **berbeda di setiap node** pada bagian `node.id` dan `advertised.listeners`. Perhatikan placeholder `<NODE_ID>` dan `<NODE_IP>`.

Buat file config baru:

```bash
sudo nano /opt/kafka/config/kraft/server.properties
```

### Node 1 (`10.0.45.98`)

```properties
# ========================= KRaft Mode Settings =========================

# Unique ID for this node in the cluster
node.id=1

# This server acts as both controller and broker
process.roles=broker,controller

# Controller quorum voters: all 3 controller nodes
controller.quorum.voters=1@10.0.45.98:9093,2@10.0.45.99:9093,3@10.0.45.100:9093

# ========================= Socket Server Settings =========================

# Listeners: broker on 9092, controller on 9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093

# Advertised listeners (hanya broker listener, JANGAN include CONTROLLER)
advertised.listeners=PLAINTEXT://10.0.45.98:9092

# Mapping listener ke security protocol
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

# Controller listener name
controller.listener.names=CONTROLLER

# Inter-broker listener
inter.broker.listener.name=PLAINTEXT

# Thread configuration (sesuai config asli)
num.network.threads=3
num.io.threads=8

# Socket buffer (sesuai config asli)
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# ========================= Log / Data Settings =========================

# Data directories (sesuai config asli)
log.dirs=/data/kafka-logs

# KRaft metadata directory
metadata.log.dir=/data/kraft-metadata

# Partitions & replication (sesuai config asli)
num.partitions=4
default.replication.factor=3
min.insync.replicas=2

# Recovery threads
num.recovery.threads.per.data.dir=1

# ========================= Internal Topic Settings =========================

offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

# ========================= Log Retention Policy =========================

# Retain logs for 7 days (sesuai config asli: 168 hours)
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

# ========================= Group Coordinator Settings =========================

# Production setting: 3 seconds delay for rebalance
group.initial.rebalance.delay.ms=3000

# ========================= Message Size (Custom FASIK) =========================

message.max.bytes=26214400
replica.fetch.max.bytes=10485760

# ========================= Production Tuning =========================

# Auto-create topics disabled (production best practice)
auto.create.topics.enable=false

# Unclean leader election disabled (prevent data loss)
unclean.leader.election.enable=false

# Controlled shutdown
controlled.shutdown.enable=true
controlled.shutdown.max.retries=3

# Replica lag
replica.lag.time.max.ms=30000

# Compression
compression.type=producer

# Log cleaner
log.cleaner.enable=true
```

### Node 2 (`10.0.45.99`) — Hanya ubah 2 baris:

```properties
node.id=2
advertised.listeners=PLAINTEXT://10.0.45.99:9092
```

### Node 3 (`10.0.45.100`) — Hanya ubah 2 baris:

```properties
node.id=3
advertised.listeners=PLAINTEXT://10.0.45.100:9092
```

> [!TIP]
> Copy file config dari Node 1 ke Node 2 & 3, lalu hanya ubah `node.id` dan `advertised.listeners`.

---

## STEP 3 — Generate Cluster ID & Format Storage

### 3.1 Generate Cluster ID (HANYA di 1 node, misal Node 1)

```bash
KAFKA_CLUSTER_ID=$(/opt/kafka/bin/kafka-storage.sh random-uuid)
echo $KAFKA_CLUSTER_ID
# Catat output-nya, contoh: MkU3OEVBNTcwNTJENDM2Qk
```

> [!CAUTION]
> **Simpan Cluster ID ini!** Harus dipakai ID yang **sama persis** di ketiga node. Jika berbeda, cluster tidak bisa terbentuk.

### 3.2 Format Storage (di SETIAP node, pakai Cluster ID yang sama)

```bash
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
  -t <CLUSTER_ID_DARI_STEP_3.1> \
  -c /opt/kafka/config/kraft/server.properties
```

Contoh:

```bash
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
  -t MkU3OEVBNTcwNTJENDM2Qk \
  -c /opt/kafka/config/kraft/server.properties
```

Output yang diharapkan:

```
Formatting /data/kafka-logs with metadata.version 3.9-IV0.
Formatting /data/kraft-metadata with metadata.version 3.9-IV0.
```

---

## STEP 4 — Buat Systemd Service

### 4.1 Buat Environment File

```bash
sudo nano /opt/kafka/config/kafka-env.sh
```

```bash
# Kafka JVM Settings - Production Tuning
# Sesuaikan heap size dengan RAM server (contoh: 6GB heap untuk server 16GB RAM)
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
export KAFKA_JVM_PERFORMANCE_OPTS="-server \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:+ExplicitGCInvokesConcurrent \
  -XX:MaxInlineLevel=15 \
  -Djava.awt.headless=true"
export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:/opt/kafka/config/log4j.properties"
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
```

```bash
sudo chown kafka:kafka /opt/kafka/config/kafka-env.sh
sudo chmod 644 /opt/kafka/config/kafka-env.sh
```

> [!TIP]
> **Panduan ukuran heap:**
> | RAM Server | KAFKA_HEAP_OPTS |
> |---|---|
> | 8 GB | `-Xms4g -Xmx4g` |
> | 16 GB | `-Xms6g -Xmx6g` |
> | 32 GB | `-Xms8g -Xmx8g` |
> | 64 GB+ | `-Xms12g -Xmx12g` |
>
> Jangan melebihi 12GB heap; sisanya biar dipakai OS untuk page cache (sangat penting untuk performa Kafka).

### 4.2 Buat Systemd Unit File

```bash
sudo nano /etc/systemd/system/kafka.service
```

```ini
[Unit]
Description=Apache Kafka Server (KRaft Mode)
Documentation=https://kafka.apache.org/documentation/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=kafka
Group=kafka
EnvironmentFile=/opt/kafka/config/kafka-env.sh

ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh

# Restart policy
Restart=on-failure
RestartSec=10

# Limits
LimitNOFILE=131072
LimitNPROC=131072

# Timeout
TimeoutStartSec=180
TimeoutStopSec=120

# OOM handling
OOMScoreAdjust=-500

# Security hardening
ProtectSystem=full
NoNewPrivileges=true
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

### 4.3 Enable & Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable kafka
```

---

## STEP 5 — OS Tuning (Production)

### 5.1 Sysctl Settings

```bash
sudo nano /etc/sysctl.d/99-kafka.conf
```

```ini
# ---- Network ----
net.core.wmem_max=2097152
net.core.rmem_max=2097152
net.ipv4.tcp_wmem=4096 65536 2048000
net.ipv4.tcp_rmem=4096 65536 2048000
net.core.netdev_max_backlog=5000
net.ipv4.tcp_max_syn_backlog=4096
net.core.somaxconn=4096

# ---- Virtual Memory ----
vm.swappiness=1
vm.dirty_ratio=60
vm.dirty_background_ratio=5
vm.max_map_count=262144

# ---- File System ----
fs.file-max=1000000
```

```bash
sudo sysctl --system
```

### 5.2 Limits untuk User Kafka

```bash
sudo nano /etc/security/limits.d/kafka.conf
```

```ini
kafka soft nofile 131072
kafka hard nofile 131072
kafka soft nproc  131072
kafka hard nproc  131072
```

### 5.3 Disable Swap (Recommended)

```bash
sudo swapoff -a
# Komentar/hapus baris swap di fstab agar permanen
sudo sed -i '/swap/d' /etc/fstab
```

---

## STEP 6 — Firewall

```bash
# Port broker (client communication)
sudo ufw allow from 10.0.45.0/24 to any port 9092 proto tcp comment "Kafka Broker"

# Port controller (internal cluster communication)
sudo ufw allow from 10.0.45.0/24 to any port 9093 proto tcp comment "Kafka Controller"

# Reload
sudo ufw reload
sudo ufw status
```

> [!NOTE]
> Jika client Kafka (producer/consumer) berada di subnet berbeda, tambahkan rule UFW untuk port `9092` dari subnet tersebut.

---

## STEP 7 — Start Cluster

> [!IMPORTANT]
> Start **semua node hampir bersamaan** (dalam jarak 1-2 menit). Controller quorum membutuhkan majority (2 dari 3 node) untuk memilih leader.

### 7.1 Start Kafka di setiap node

```bash
# Node 1
sudo systemctl start kafka

# Node 2
sudo systemctl start kafka

# Node 3
sudo systemctl start kafka
```

### 7.2 Cek Status

```bash
sudo systemctl status kafka
```

### 7.3 Cek Log

```bash
# Lihat log real-time
sudo journalctl -u kafka -f

# Atau langsung dari file log Kafka
tail -f /opt/kafka/logs/server.log
```

Output yang menandakan sukses:

```
[KafkaServer id=1] started (kafka.server.KafkaServer)
```

---

## STEP 8 — Verifikasi Cluster

### 8.1 Cek Metadata Cluster

```bash
/opt/kafka/bin/kafka-metadata.sh --snapshot /data/kraft-metadata/__cluster_metadata-0/00000000000000000000.log --cluster-id
```

### 8.2 Cek Broker yang Terdaftar

```bash
/opt/kafka/bin/kafka-broker-api-versions.sh \
  --bootstrap-server 10.0.45.98:9092,10.0.45.99:9092,10.0.45.100:9092
```

### 8.3 Buat Topic Test

```bash
/opt/kafka/bin/kafka-topics.sh --create \
  --topic test-kraft \
  --partitions 4 \
  --replication-factor 3 \
  --bootstrap-server 10.0.45.98:9092,10.0.45.99:9092,10.0.45.100:9092
```

### 8.4 Describe Topic

```bash
/opt/kafka/bin/kafka-topics.sh --describe \
  --topic test-kraft \
  --bootstrap-server 10.0.45.98:9092
```

Output yang diharapkan:

```
Topic: test-kraft    TopicId: xxxx    PartitionCount: 4    ReplicationFactor: 3
    Partition: 0    Leader: 1    Replicas: 1,2,3    Isr: 1,2,3
    Partition: 1    Leader: 2    Replicas: 2,3,1    Isr: 2,3,1
    Partition: 2    Leader: 3    Replicas: 3,1,2    Isr: 3,1,2
    Partition: 3    Leader: 1    Replicas: 1,3,2    Isr: 1,3,2
```

> [!TIP]
> Pastikan **ISR (In-Sync Replicas)** sama dengan **Replicas** di semua partition. Jika ada yang berbeda, ada node yang belum fully synced.

### 8.5 Test Produce & Consume

```bash
# Terminal 1: Produce
echo "Hello KRaft Kafka" | /opt/kafka/bin/kafka-console-producer.sh \
  --topic test-kraft \
  --bootstrap-server 10.0.45.98:9092

# Terminal 2: Consume
/opt/kafka/bin/kafka-console-consumer.sh \
  --topic test-kraft \
  --from-beginning \
  --bootstrap-server 10.0.45.98:9092
```

### 8.6 Hapus Topic Test (setelah verifikasi selesai)

```bash
/opt/kafka/bin/kafka-topics.sh --delete \
  --topic test-kraft \
  --bootstrap-server 10.0.45.98:9092
```

---

## STEP 9 — Monitoring (Optional tapi Recommended)

### 9.1 Enable JMX

Tambahkan di `/opt/kafka/config/kafka-env.sh`:

```bash
export JMX_PORT=9999
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false \
  -Dcom.sun.management.jmxremote.local.only=true \
  -Djava.rmi.server.hostname=localhost"
```

### 9.2 Health Check Script

```bash
sudo nano /opt/kafka/bin/kafka-healthcheck.sh
```

```bash
#!/bin/bash
# Simple Kafka health check
BOOTSTRAP="localhost:9092"

# Check if Kafka process is running
if ! systemctl is-active --quiet kafka; then
    echo "CRITICAL: Kafka service is not running"
    exit 2
fi

# Check if broker is responding
if /opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server $BOOTSTRAP > /dev/null 2>&1; then
    echo "OK: Kafka broker is responding"
    exit 0
else
    echo "WARNING: Kafka broker is not responding"
    exit 1
fi
```

```bash
sudo chmod +x /opt/kafka/bin/kafka-healthcheck.sh
sudo chown kafka:kafka /opt/kafka/bin/kafka-healthcheck.sh
```

---

## STEP 10 — Log Rotation

```bash
sudo nano /etc/logrotate.d/kafka
```

```
/opt/kafka/logs/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

---

## Perbedaan Config dari Project Asli

| Setting | Config Asli (ZooKeeper) | Config Baru (KRaft) | Alasan |
|---------|------------------------|---------------------|--------|
| `zookeeper.connect` | ✅ Ada | ❌ Dihapus | KRaft tidak butuh ZooKeeper |
| `broker.id` | `broker.id=1` | `node.id=1` | Nama property berubah di KRaft |
| `process.roles` | Tidak ada | `broker,controller` | Combined mode KRaft |
| `controller.quorum.voters` | Tidak ada | `1@...:9093,...` | Menggantikan ZooKeeper ensemble |
| `controller.listener.names` | Tidak ada | `CONTROLLER` | Listener khusus controller |
| `metadata.log.dir` | Tidak ada | `/data/kraft-metadata` | Menggantikan ZooKeeper data dir |
| `transaction.state.log.min.isr` | `3` | `2` | Lebih aman; ISR=3 terlalu ketat untuk 3-node cluster |
| `auto.create.topics.enable` | Default (true) | `false` | Production best practice |
| `unclean.leader.election.enable` | Default (false) | `false` (explicit) | Mencegah data loss |

---

## Troubleshooting

### Kafka tidak mau start

```bash
# Cek log
sudo journalctl -u kafka --no-pager -n 100

# Cek permission
ls -la /data/kafka-logs/
ls -la /data/kraft-metadata/

# Cek port conflict
sudo ss -tlnp | grep -E '9092|9093'
```

### Controller quorum tidak terbentuk

```bash
# Pastikan ketiga node bisa saling reach port 9093
# Dari Node 1:
nc -zv 10.0.45.99 9093
nc -zv 10.0.45.100 9093
```

### Error "Cluster ID mismatch"

```bash
# HATI-HATI: ini akan menghapus semua data!
sudo systemctl stop kafka
sudo rm -rf /data/kafka-logs/*
sudo rm -rf /data/kraft-metadata/*

# Format ulang dengan Cluster ID yang benar
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
  -t <CLUSTER_ID_YANG_BENAR> \
  -c /opt/kafka/config/kraft/server.properties

sudo systemctl start kafka
```

### Out of Memory

```bash
# Turunkan heap size di /opt/kafka/config/kafka-env.sh
# Cek memory usage
free -h
```

---

## Quick Reference — Perintah Berguna

```bash
# Start/Stop/Restart
sudo systemctl start kafka
sudo systemctl stop kafka
sudo systemctl restart kafka

# Status
sudo systemctl status kafka

# Logs
sudo journalctl -u kafka -f

# List topics
/opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# Describe topic
/opt/kafka/bin/kafka-topics.sh --describe --topic <TOPIC_NAME> --bootstrap-server localhost:9092

# Consumer groups
/opt/kafka/bin/kafka-consumer-groups.sh --list --bootstrap-server localhost:9092

# Describe consumer group
/opt/kafka/bin/kafka-consumer-groups.sh --describe --group <GROUP_NAME> --bootstrap-server localhost:9092

# Cluster metadata
/opt/kafka/bin/kafka-metadata.sh --snapshot /data/kraft-metadata/__cluster_metadata-0/00000000000000000000.log --cluster-id
```
