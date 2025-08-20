# PostgreSQL Remote Connection Setup Guide

## 1. Configure PostgreSQL to Accept Remote Connections

Edit PostgreSQL configuration:

```bash
sudo nano /etc/postgresql/[version]/main/postgresql.conf
```

Set these parameters:

```
listen_addresses = '*'
port = 5432
```

## 2. Configure Client Authentication

Edit authentication file:

```bash
sudo nano /etc/postgresql/[version]/main/pg_hba.conf
```

Add connection rules:

```
# Local connections
local   all             postgres                                peer
host    all             all             127.0.0.1/32            md5

# Remote connection from specific IP
host    your_db         your_user       YOUR_IP_ADDRESS/32      md5

# Example:
host    database          user    ip/32       md5
```

## 3. Restart PostgreSQL

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

## 4. Configure Firewall

Allow PostgreSQL port from your IP only:

```bash
# UFW
sudo ufw allow from YOUR_IP_ADDRESS to any port 5432

# iptables (alternative)
sudo iptables -A INPUT -p tcp -s YOUR_IP_ADDRESS --dport 5432 -j ACCEPT
```

## 5. Create User and Database (if needed)

```bash
sudo -u postgres psql
```

```sql
CREATE USER your_user WITH PASSWORD 'secure_password';
CREATE DATABASE your_db OWNER your_user;
GRANT ALL PRIVILEGES ON DATABASE your_db TO your_user;
\q
```

## 6. Connect Remotely

From your local machine:

```bash
psql -h SERVER_IP -U your_user -d your_db
```

## 7. Optional: Enable SSL (Recommended)

In `postgresql.conf`:

```
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
```

In `pg_hba.conf`, change `host` to `hostssl`:

```
hostssl your_db         your_user       YOUR_IP_ADDRESS/32      md5
```

Connect with SSL:

```bash
psql "host=SERVER_IP port=5432 dbname=your_db user=your_user sslmode=require"
```

## Quick Troubleshooting

- **Check if PostgreSQL is listening**: `sudo netstat -tlnp | grep :5432`
- **View logs**: `sudo tail -f /var/log/postgresql/postgresql-*-main.log`
- **Test port connectivity**: `telnet SERVER_IP 5432`
- **Check firewall**: `sudo ufw status`