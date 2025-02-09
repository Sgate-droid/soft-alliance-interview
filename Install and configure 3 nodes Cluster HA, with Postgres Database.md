Setting up a 3-node High Availability (HA) cluster with PostgreSQL on Ubuntu involves several steps. Below is a step-by-step guide to help you through the process.

### Prerequisites
1. **Three Ubuntu Servers**: Ensure you have three Ubuntu servers (20.04 or 22.04) with static IP addresses.
2. **SSH Access**: Ensure you have SSH access to all three servers.
3. **Sudo Privileges**: Ensure you have sudo privileges on all servers.
4. **Network Configuration**: Ensure that the servers can communicate with each other over the network.

### Step 1: Update and Upgrade All Servers
First, update and upgrade all three servers to ensure they have the latest packages.

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2: Install PostgreSQL on All Nodes
Install PostgreSQL on all three nodes.

```bash
sudo apt install postgresql postgresql-contrib -y
```

### Step 3: Configure PostgreSQL on All Nodes
1. **Edit `pg_hba.conf`**: Allow replication connections between nodes.

   ```bash
   sudo nano /etc/postgresql/<version>/main/pg_hba.conf
   ```

   Add the following lines (replace `<version>` with your PostgreSQL version, e.g., `14`):

   ```plaintext
   host    replication     replicator      <node1_ip>/32        md5
   host    replication     replicator      <node2_ip>/32        md5
   host    replication     replicator      <node3_ip>/32        md5
   ```

2. **Edit `postgresql.conf`**: Configure PostgreSQL to listen on all interfaces.

   ```bash
   sudo nano /etc/postgresql/<version>/main/postgresql.conf
   ```

   Modify the following lines:

   ```plaintext
   listen_addresses = '*'
   wal_level = replica
   max_wal_senders = 3
   wal_keep_segments = 32
   hot_standby = on
   ```

3. **Restart PostgreSQL**:

   ```bash
   sudo systemctl restart postgresql
   ```

### Step 4: Set Up Replication
1. **Create a Replication User**: On the primary node (Node 1), create a replication user.

   ```bash
   sudo -u postgres psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'your_password';"
   ```

2. **Backup the Primary Node**: On Node 1, create a base backup for replication.

   ```bash
   sudo -u postgres pg_basebackup -h <node1_ip> -D /var/lib/postgresql/<version>/main/ -U replicator -P -v -R -X stream -C -S standby1
   ```

3. **Copy the Backup to Standby Nodes**: Copy the backup to Node 2 and Node 3.

   ```bash
   rsync -avz /var/lib/postgresql/<version>/main/ <node2_ip>:/var/lib/postgresql/<version>/main/
   rsync -avz /var/lib/postgresql/<version>/main/ <node3_ip>:/var/lib/postgresql/<version>/main/
   ```

4. **Configure Standby Nodes**: On Node 2 and Node 3, edit the `recovery.conf` file (or `postgresql.auto.conf` in newer versions).

   ```bash
   sudo nano /var/lib/postgresql/<version>/main/recovery.conf
   ```

   Add the following lines:

   ```plaintext
   standby_mode = 'on'
   primary_conninfo = 'host=<node1_ip> port=5432 user=replicator password=your_password'
   trigger_file = '/tmp/postgresql.trigger'
   ```

5. **Start PostgreSQL on Standby Nodes**: Start PostgreSQL on Node 2 and Node 3.

   ```bash
   sudo systemctl start postgresql
   ```

### Step 5: Test Replication
1. **Check Replication Status**: On the primary node (Node 1), check the replication status.

   ```bash
   sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"
   ```

2. **Create a Test Database**: On Node 1, create a test database and verify it replicates to Node 2 and Node 3.

   ```bash
   sudo -u postgres psql -c "CREATE DATABASE testdb;"
   ```

   Verify on Node 2 and Node 3:

   ```bash
   sudo -u postgres psql -c "\l"
   ```

### Step 6: Configure HA with Pgpool-II
1. **Install Pgpool-II**: Install Pgpool-II on all three nodes.

   ```bash
   sudo apt install pgpool2 -y
   ```

2. **Configure Pgpool-II**: Edit the Pgpool-II configuration file.

   ```bash
   sudo nano /etc/pgpool2/pgpool.conf
   ```

   Modify the following lines:

   ```plaintext
   listen_addresses = '*'
   backend_hostname0 = '<node1_ip>'
   backend_port0 = 5432
   backend_weight0 = 1
   backend_hostname1 = '<node2_ip>'
   backend_port1 = 5432
   backend_weight1 = 1
   backend_hostname2 = '<node3_ip>'
   backend_port2 = 5432
   backend_weight2 = 1
   ```

3. **Start Pgpool-II**: Start Pgpool-II on all nodes.

   ```bash
   sudo systemctl start pgpool2
   ```

### Step 7: Test HA Setup
1. **Connect via Pgpool-II**: Connect to the PostgreSQL cluster via Pgpool-II.

   ```bash
   psql -h <pgpool_ip> -U postgres
   ```

2. **Failover Test**: Simulate a failure on Node 1 and verify that Node 2 or Node 3 takes over.

   ```bash
   sudo systemctl stop postgresql
   ```

   Verify that you can still connect to the database via Pgpool-II.

### Step 8: Automate Failover with Scripts
1. **Create Failover Script**: Create a failover script to automate the failover process.

   ```bash
   sudo nano /etc/pgpool2/failover.sh
   ```

   Add the following content:

   ```bash
   #!/bin/bash
   FAILED_NODE=$1
   NEW_MASTER=$2
   psql -h $NEW_MASTER -U postgres -c "SELECT pg_promote(true);"
   ```

2. **Make the Script Executable**:

   ```bash
   sudo chmod +x /etc/pgpool2/failover.sh
   ```

3. **Configure Pgpool-II to Use the Script**: Edit the Pgpool-II configuration file.

   ```bash
   sudo nano /etc/pgpool2/pgpool.conf
   ```

   Add the following lines:

   ```plaintext
   failover_command = '/etc/pgpool2/failover.sh %d %h'
   ```

4. **Restart Pgpool-II**:

   ```bash
   sudo systemctl restart pgpool2
   ```

### Step 9: Monitor the Cluster
1. **Install Monitoring Tools**: Install monitoring tools like `pg_stat_activity` or third-party tools like `pgAdmin` or `Zabbix`.

2. **Set Up Alerts**: Configure alerts for any failures or performance issues.

### Conclusion
You have now set up a 3-node High Availability PostgreSQL cluster with Pgpool-II on Ubuntu. This setup ensures that your database remains available even if one of the nodes fails. Regularly monitor the cluster and test failover scenarios to ensure everything is working as expected.

### Additional Tips
- **Backup**: Regularly back up your database.
- **Security**: Ensure your PostgreSQL and Pgpool-II configurations are secure.
- **Documentation**: Keep documentation of your setup for future reference.

This guide provides a basic setup. Depending on your specific requirements, you may need to adjust configurations or add additional components like load balancers or more advanced monitoring tools.
