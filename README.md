# MySQL Replication with ProxySQL

This project is a hands-on example showing how to make your backend database faster, more reliable, and easier to scale using MySQL replication and ProxySQL, all managed with Docker Compose. It uses:
 
* One MySQL master (for writes)
* Two MySQL slaves (for reads)
* ProxySQL (for smart query routing and load balancing)
* FastAPI (as a sample CRUD backend)
* Docker Compose (for easy orchestration)
It’s designed to show how to scale the database for high read-loads, ensure high availability, and keep your application code simple.

Why Replication of slave db is required?
* Performance: Distributing read queries across multiple servers.
* Reliability: If a slave fails, others can still serve reads.
* Scalability: Adding more slaves when user base grows.
* Separation of Concerns: Writes go to the master, reads go to slaves.

## Architecture
<img src="graph/m8.svg" alt="Implementation Diagram" width="1000">

### **How It Works:**

1. *User* interacts with the FastAPI CRUD app (e.g., via browser or Postman).Sends HTTP requests (GET, POST, etc.)
2. *FastAPI* Receives requests, talks to ProxySQL and sends all DB queries to ProxySQL.
3. *ProxySQL*Routes queries to the right MySQL server.

> *Write queries* (INSERT, UPDATE, DELETE) → sent to *MySQL Master*

> *Read queries* (SELECT) → load-balanced across *MySQL Slaves*

4. *MySQL Master* logs all changes and replicates them to the slaves.
5. *MySQL Slaves* stay in sync and serve read requests.


## Workflow Diagram:
<div align="center">
<img src="graph/flow.svg" alt="Implementation Diagram" width="500" height="1000">
</div>


## Project Structure

```

mysql-replication-poc/
├── docker-compose.yml                            # Orchestrates all services
│
├── master/                                       # Master MySQL container
│ ├── my.cnf                                      # Master MySQL configuration
│ └── init.sql                                    # Initial SQL for schema/data
│
├── slave/                                        # Slave MySQL containers
│ ├── my-slave1.cnf                               # Slave 1 configuration
│ ├── my-slave2.cnf                               # Slave 2 configuration
│ └── init.sql                                    # Initial SQL script for slaves
│
├── proxysql/                                     # ProxySQL container
│ └── proxysql.cnf                                # ProxySQL configuration
│
├── fastapi-app/                                  # FastAPI CRUD app
│ └── ... # App source code, Dockerfile, requirements.txt
│
├── graph                              # Architecture diagram
└── README.md                                    # This file
```



## Configuration Details
1. MySQL Master (master/my.cnf)
``` bash
[mysqld]
server-id = 1
log_bin = /var/run/mysqld/mysql-bin.log
binlog_do_db = sbtest
gtid_mode = on
enforce_gtid_consistency = on
log_slave_updates = on
```
 
2. MySQL Slaves (slave/my-slave1.cnf, slave/my-slave2.cnf)
```bash
[mysqld]
server-id = 2  # (or 3 for slave2)
relay-log = /var/run/mysqld/mysql-relay-bin.log
read_only = on
gtid_mode = on
enforce_gtid_consistency = on
log_slave_updates = on
```


3. Replication Setup (master/init.sql, slave/init.sql)

* Master creates a replication user and a sample table.
* Slaves connect to the master and start replication.

4. ProxySQL (proxysql/proxysql.cnf)
```bash
mysql_servers = (
    { address="mysql-master" , port=3306 , hostgroup=10 },
    { address="mysql-slave1" , port=3306 , hostgroup=20 },
    { address="mysql-slave2" , port=3306 , hostgroup=20 }
)
mysql_query_rules = (
    { rule_id=200, match_pattern="^SELECT .*", destination_hostgroup=20, apply=1 },
    { rule_id=300, match_pattern=".*", destination_hostgroup=10, apply=1 }
)

```

* Hostgroup 10: Master (writes)
* Hostgroup 20: Slaves (reads)
* Rules:
>> SELECT → slaves
>> Everything else → master


5. FastAPI App (fastapi-app/)

* Connects to ProxySQL, not directly to MySQL.
* Handles CRUD operations for a users table.



## Poridhi Lab Setup & Run
### 1. Clone the repository

```bash
git clone https://github.com/poridhioss/mysql-replication-poc.git
cd mysql-replication-poc
```

### 2. Start all services
```bash
docker-compose up -d
```

### 3. Check containers
```bash
docker ps
```
wait when all status are healty:
![image](https://github.com/user-attachments/assets/d18ec046-e2cb-4290-bd0d-9026295fbe87)


### 4. *Access the application through load balancer*

Get IP from eth0 using:
```bash
sudo apt update
apt install iproute2 -y
ip addr show eth0
```

<div align="center">
  <img src="https://github.com/user-attachments/assets/1832af97-3cb3-4faf-8dd0-d6d85cd84dce" >
</div>


Create load balancer and configure with your application's IP and port in Poridhi lab:
<div align="center">
  <img src="https://github.com/user-attachments/assets/b50a7d65-6fc0-4a96-92dc-0fb8ad333cbf">
</div>

![WhatsApp Image 2025-07-03 at 19 45 21_3f2e82c9](https://github.com/user-attachments/assets/5316afb9-eadb-49d4-b093-1dfa7684228c)

Brower:


![image](https://github.com/user-attachments/assets/59b30c21-9688-4343-a265-ee045c5df9d6)


### 5.Test the FastAPI app

Open http://<provided_URL>:8001/docs for the FastAPI Swagger UI:

![image](https://github.com/user-attachments/assets/ad01e76a-86cb-4210-a5f8-0fc201da3dca)

then Execute CRUD operations and check the logs of each services.

### 6 .Verify ProxySQL Configuration
Ensure ProxySQL is routing queries correctly and all servers are online.

6.1 Install MySQL Client
Install a MySQL client to interact with ProxySQL.
```bash 
sudo apt update
sudo apt install mysql-client -y
```

6.2 Check ProxySQL Server Status
Verify that ProxySQL recognizes all MySQL servers.
```bash
mysql -h 0.0.0.0 -P 6032 -u admin2 -ppass2 -e 'SELECT * FROM mysql_servers;'
```
Expected Output:
![image](https://github.com/user-attachments/assets/6c05079e-60b6-4a44-babd-03e17614ceba)




### 7.Monitor Startup Process and Complete Testing Workflow

7.1. Check Master Logs (Write Operations):

Real-time Master Log Monitoring:
```bash 
# Monitor master logs in real-time
docker-compose exec mysql-master sh -c 'tail -f /var/log/mysql/general.log'
```
output:
![image](https://github.com/user-attachments/assets/e427fc7f-7c8d-4ad1-88d4-e617f2fea72e)

when a write operation is made (e.g a new user is creaated via. swagger UI):
![image](https://github.com/user-attachments/assets/87c5e80f-1c57-49a2-853e-043fcca6c7cf)

this Insertation can be seen in the sql-master logs:
![image](https://github.com/user-attachments/assets/0b0a4fd0-fc78-4efc-a227-c382f0bfaa89)

while the sql-slaves sits idle.

Check Master Binary Log (Replication Events):
```bash
# Show binary log files
docker-compose exec mysql-master sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'SHOW BINARY LOGS'"

# Show current binary log events
docker-compose exec mysql-master sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'SHOW BINLOG EVENTS'"

# Show master status (current log file and position)
docker-compose exec mysql-master sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'SHOW MASTER STATUS\G'"

```
![image](https://github.com/user-attachments/assets/32641634-dcac-4dbd-8ff8-6c4d8f70faa9)


7.2. Check Slave Logs (Replication Changes)

when read operation is done e.g to see data of  specific user or list of all users in the data-base, the slave db are activated.
![image](https://github.com/user-attachments/assets/00cd65f2-d0c9-4a43-8825-e91790fbda3a)

Real-time Slave Log Monitoring:
```bash
# Monitor slave1 logs in real-time
docker-compose exec mysql-slave1 sh -c 'tail -f /var/log/mysql/general.log'

# Monitor slave2 logs in real-time
docker-compose exec mysql-slave2 sh -c 'tail -f /var/log/mysql/general.log'
```

in read operation the work-load is distributed between the slaves which can be viewed from there logs:

![image](https://github.com/user-attachments/assets/a4174d3a-8130-4968-b992-b2911d8dd545)

in these case the master-sql sits idle:
<div align="center">

![image](https://github.com/user-attachments/assets/d9006ab9-ee1f-4ac4-8819-8b2285224fea)
</div>


now again when a user is deleted / data is modified the master-sql is activated:

![image](https://github.com/user-attachments/assets/9571dd9a-f693-4765-bcff-2c1ed12c1094)


output:

![image](https://github.com/user-attachments/assets/87ddab44-2de5-4faa-8a4d-9901f77b3e8a)



## Overall Quick Status Check Commands
Step 1: Open Multiple Terminal Windows
```bash
cd mysql-replication-poc/
```
<div align="center">
![image](https://github.com/user-attachments/assets/89823257-2ed7-4e8a-8ae4-71e9f8d16ca7)
</div>
Terminal 1 - Master Logs:
```bash
docker-compose exec mysql-master sh -c 'tail -f /var/log/mysql/general.log'
```

Terminal 2 - Slave1 Logs:
```bash
docker-compose exec mysql-slave1 sh -c 'tail -f /var/log/mysql/general.log'
```
Terminal 3 - Slave2 Logs:
```bash
docker-compose exec mysql-slave2 sh -c 'tail -f /var/log/mysql/general.log'
```
you can also perform the operations from the terminal:

Create a new user (POST request):
```bash
curl -X POST "http://localhost:8001/users/" \
     -H "Content-Type: application/json" \
     -d '{"name": "John Doe"}'
```

![image](https://github.com/user-attachments/assets/88b9b9f1-5865-423f-81c6-829f0cbba384)

 Observe the Logs ;What you'll see in Master Logs:

![image](https://github.com/user-attachments/assets/e9f33b8d-42d0-4dbf-9431-3345cf10fa2e)


## Conclusion
This lab demonstrated how to set up a MySQL master-slave replication cluster with ProxySQL for intelligent query routing and load balancing. By using Docker Compose, you created a scalable, high-availability database architecture with one master for writes, two slaves for reads, and ProxySQL for efficient query distribution. The setup includes automated replication with GTID, health checks, and monitoring, making it suitable for production environments.








