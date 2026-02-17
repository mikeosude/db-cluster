# Step-by-Step: Percona XtraDB Cluster 8.0 with Step-CA TLS (Galera)

1. Define your nodes

Example:

| Node | Hostname        | IP         |
|------|-----------------|------------|
| db1  | db1.example.com | 10.10.0.62 |
| db2  | db2.example.com | 10.10.0.63 |
| db3  | db3.example.com | 10.10.0.64 |

Optional VIP / shared DNS:

- db.example.com → 10.10.0.249

⸻

1. Set host resolution on all nodes

Edit /etc/hosts on db1, db2, db3:
```sh
sudo tee -a /etc/hosts >/dev/null <<'EOF'
10.10.0.62 db1.example.com db1
10.10.0.63 db2.example.com db2
10.10.0.64 db3.example.com db3
10.10.0.249 db.example.com
EOF
```
Validate:
```sh
getent hosts db1 db2 db3 db.example.com
```

⸻

3. Firewall configuration (firewalld) on all nodes

Add only the cluster node IPs to the trusted zone:
```sh
sudo firewall-cmd --permanent --zone=trusted --add-source=10.10.0.6{2,3,4}
sudo firewall-cmd --reload
```
Confirm:
```sh
sudo firewall-cmd --zone=trusted --list-sources
```

⸻

4. Install Percona XtraDB Cluster on all nodes
```sh
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release setup pxc-80
sudo dnf install -y percona-xtradb-cluster
sudo systemctl enable mysql
```

⸻

5. Generate a private key + CSR config

On one node (recommended: db1), create a CSR config file pxc.cnf:
```ini
[ req ]
default_bits       = 4096
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext

[ dn ]
CN = db.example.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = db1.example.com
DNS.2 = db2.example.com
DNS.3 = db3.example.com

DNS.4 = db1
DNS.5 = db2
DNS.6 = db3

IP.1  = 10.10.0.62
IP.2  = 10.10.0.63
IP.3  = 10.10.0.64
IP.4  = 10.10.0.249
```

Generate key + CSR:

```sh
openssl req -new -nodes \
  -newkey rsa:4096 \
  -keyout server-key.pem \
  -out pxc.csr \
  -config pxc.cnf
```

⸻

6. Issue a 10-year certificate from Step-CA

Using the step CLI:
```sh
step ca certificate "db.example.com" \
  server-cert.pem server-key.pem \
  --san db1.example.com \
  --san db2.example.com \
  --san db3.example.com \
  --san db1 \
  --san db2 \
  --san db3 \
  --san 10.10.0.62 \
  --san 10.10.0.63 \
  --san 10.10.0.64 \
  --san 10.10.0.249 \
  --not-after=87600h
```
Copy/export your Step-CA root certificate to a file named:
	•	ca.pem

⸻

7. Copy certificates into MySQL data directory on ALL nodes

On each node, place:
	•	ca.pem
	•	server-cert.pem
	•	server-key.pem

into:
	•	/var/lib/mysql/

Example:

```sh
sudo cp ca.pem /var/lib/mysql/ca.pem
sudo cp server-cert.pem /var/lib/mysql/server-cert.pem
sudo cp server-key.pem /var/lib/mysql/server-key.pem

sudo chown mysql:mysql /var/lib/mysql/ca.pem /var/lib/mysql/server-cert.pem /var/lib/mysql/server-key.pem
sudo chmod 644 /var/lib/mysql/ca.pem /var/lib/mysql/server-cert.pem
sudo chmod 600 /var/lib/mysql/server-key.pem
```

Verify the certificate chain on each node:
```sh
openssl verify -CAfile /var/lib/mysql/ca.pem /var/lib/mysql/server-cert.pem
```
Verify files match across nodes (hashes must match):
```sh
sha256sum /var/lib/mysql/ca.pem /var/lib/mysql/server-cert.pem /var/lib/mysql/server-key.pem
```

⸻

8. Configure /etc/my.cnf on each node

Use a single my.cnf configuration.

db1 /etc/my.cnf
```ini
[client]
socket=/var/lib/mysql/mysql.sock

[mysqld]
server-id=1
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

binlog_expire_logs_seconds=604800
binlog_format=ROW
innodb_autoinc_lock_mode=2

wsrep_provider=/usr/lib64/galera4/libgalera_smm.so
wsrep_cluster_name=mikeosude-cluster
wsrep_cluster_address=gcomm://10.10.0.62,10.10.0.63,10.10.0.64
wsrep_node_address=10.10.0.62
wsrep_node_name=db1

wsrep_slave_threads=8
pxc_strict_mode=ENFORCING
wsrep_sst_method=xtrabackup-v2

```

db2 changes
```ini
server-id=2
wsrep_node_address=10.10.0.63
wsrep_node_name=db2
```
db3 changes
```ini
server-id=3
wsrep_node_address=10.10.0.64
wsrep_node_name=db3
```

⸻

9. Bootstrap the cluster on db1

On db1:
```sh
sudo systemctl stop mysql 2>/dev/null || true
sudo pkill -9 mysqld 2>/dev/null || true
sudo systemctl start mysql@bootstrap.service
```
Confirm ports:
```sh
sudo ss -lntp | egrep ':(3306|4567)\b'
```

⸻

10) Start MySQL on db2 and db3 to join

On db2:
```sh
sudo systemctl start mysql
```
On db3:
```sh
sudo systemctl start mysql
```

⸻

11. Validate cluster status

From db1:
```sql
mysql -uroot -p -e "
SHOW STATUS LIKE 'wsrep_cluster_size';
SHOW STATUS LIKE 'wsrep_cluster_status';
SHOW STATUS LIKE 'wsrep_ready';
"
```
Expected:
	•	wsrep_cluster_size = 3
	•	wsrep_cluster_status = Primary
	•	wsrep_ready = ON

⸻

12. Confirm TLS settings are active

From any node:
```sh
mysql -uroot -p -e "SHOW VARIABLES LIKE 'wsrep_provider_options'\G" | egrep 'socket\.ssl|socket\.ssl_ca|socket\.ssl_cert|socket\.ssl_key'
```

⸻