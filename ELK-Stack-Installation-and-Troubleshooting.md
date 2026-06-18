                +----------------+
                |     Kibana     |
                +----------------+
                         |
                         |
                +----------------+
                | Elasticsearch  |
                +----------------+
                   /          \
                  /            \
                 /              \
      +----------------+  +----------------+
      |     EC2-1      |  |     EC2-2      |
      |   Filebeat     |  |   Filebeat     |
      |  Metricbeat    |  |  Metricbeat    |
      +----------------+  +----------------+ 

      ⸻

2. COMPONENTS USED

Elasticsearch

* Stores logs and metrics
* Indexing and searching engine

Kibana

* Visualization layer
* Dashboards
* Log analysis

Filebeat

* Log shipper
* Sends logs to Elasticsearch

Metricbeat

* Metrics collector
* CPU
* Memory
* Disk
* Network

⸻

3. EC2 REQUIREMENTS

Monitoring Server (EC2-1)

Recommended:

* Ubuntu 22.04
* t3.medium or higher
* 4GB RAM minimum

Ports:

22
5601
9200

Application Server (EC2-2)

* Ubuntu 22.04

Ports:

22

⸻

4. JAVA INSTALLATION

Update packages:

sudo apt update

Install Java:

sudo apt install openjdk-17-jdk -y

Verify:

java -version

Expected:

openjdk version “17”

⸻

5. ADD ELASTIC REPOSITORY

Import GPG key:

wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg –dearmor -o /usr/share/keyrings/elastic-keyring.gpg

Create repository:

echo “deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main” | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

Update:

sudo apt update

⸻

6. INSTALL ELASTICSEARCH

sudo apt install elasticsearch -y

Enable service:

sudo systemctl enable elasticsearch

⸻

7. ELASTICSEARCH CONFIGURATION

Edit:

sudo nano /etc/elasticsearch/elasticsearch.yml

Configuration:

network.host: 0.0.0.0

discovery.type: single-node

⸻

8. IMPORTANT LESSON: SINGLE NODE VS CLUSTER

Single Node

One Elasticsearch server.

Use:

discovery.type: single-node

Multi Node Cluster

Multiple Elasticsearch servers.

Use:

cluster.initial_master_nodes

and

discovery.seed_hosts

NEVER USE BOTH TOGETHER.

Incorrect:

discovery.type: single-node

cluster.initial_master_nodes:

* node-1

Result:

Elasticsearch startup failure.

This was the primary issue encountered during deployment.

⸻

9. START ELASTICSEARCH

Start:

sudo systemctl start elasticsearch

Verify:

sudo systemctl status elasticsearch

Test:

curl -k -u elastic https://localhost:9200

Expected:

JSON response containing cluster information.

⸻

10. ELASTIC SECURITY

Elastic 8.x enables security automatically.

Generated:

* TLS Certificates
* Elastic User
* Enrollment System

Therefore:

HTTPS is mandatory.

Example:

curl -k -u elastic https://localhost:9200

NOT:

curl http://localhost:9200

⸻

11. RESET ELASTIC PASSWORD

Command:

sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic

Output:

Generated password

Store securely.

Login user:

elastic

⸻

12. KIBANA INSTALLATION

Install:

sudo apt install kibana -y

Enable:

sudo systemctl enable kibana

⸻

13. KIBANA CONFIGURATION

Edit:

sudo nano /etc/kibana/kibana.yml

Set:

server.host: “0.0.0.0”

Restart:

sudo systemctl restart kibana

⸻

14. KIBANA ENROLLMENT

Generate Enrollment Token:

sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana

Token Example:

eyJ2ZXIiOiI4LjE5LjE2Iiw…

Very long token.

⸻

15. KIBANA VERIFICATION CODE

Check logs:

sudo journalctl -u kibana -f

Example:

Your verification code is:

007865

Important:

Verification Code ≠ Enrollment Token

Verification Code:
Short 6-digit number

Enrollment Token:
Long encoded string

This distinction caused significant confusion during setup.

⸻

16. ACCESS KIBANA

URL:

http://PUBLIC-IP:5601

Login:

Username:
elastic

Password:
Generated using reset-password

⸻

17. FILEBEAT INSTALLATION

On BOTH servers:

sudo apt install filebeat -y

⸻

18. FILEBEAT CONFIGURATION

Edit:

sudo nano /etc/filebeat/filebeat.yml

Configure:

output.elasticsearch:
hosts: [“https://172.31.93.223:9200”]
username: “elastic”
password: “YOUR_PASSWORD”
ssl.verification_mode: none

⸻

19. ENABLE FILEBEAT SYSTEM MODULE

Enable:

sudo filebeat modules enable system

⸻

20. CRITICAL FILEBEAT ISSUE

Error:

module system is configured but has no enabled filesets

Cause:

system.yml contained:

syslog:
enabled: false

auth:
enabled: false

Fix:

Edit:

sudo nano /etc/filebeat/modules.d/system.yml

Set:

* module: system
    syslog:
    enabled: true
    auth:
    enabled: true

After fixing:

sudo systemctl restart filebeat

⸻

21. FILEBEAT VALIDATION

Configuration Test:

sudo filebeat test config -e

Expected:

Config OK

Output Test:

sudo filebeat test output -e

Expected:

connection… OK

TLS… OK

talk to server… OK

version: 8.19.x

⸻

22. FILEBEAT TROUBLESHOOTING COMMANDS

Check service:

sudo systemctl status filebeat

Check logs:

sudo journalctl -u filebeat -n 100 –no-pager

Run foreground:

sudo filebeat -e

⸻

23. GENERATE TEST LOGS

EC2-1:

logger “HELLO FROM ELK SERVER”

EC2-2:

logger “HELLO FROM EC2-2”

⸻

24. KIBANA DISCOVER

Navigate:

Analytics
→ Discover

Create Data View:

filebeat-*

Search:

HELLO

Expected:

Logs from both servers.

⸻

25. METRICBEAT INSTALLATION

Install:

sudo apt install metricbeat -y

⸻

26. METRICBEAT CONFIGURATION

Edit:

sudo nano /etc/metricbeat/metricbeat.yml

Configure:

output.elasticsearch:
hosts: [“https://172.31.93.223:9200”]
username: “elastic”
password: “YOUR_PASSWORD”
ssl.verification_mode: none

Enable module:

sudo metricbeat modules enable system

Setup:

sudo metricbeat setup

Start:

sudo systemctl enable metricbeat

sudo systemctl restart metricbeat

⸻

27. SECURITY GROUPS

EC2-1

Port 22
Source: Your IP

Port 5601
Source: Your IP

Port 9200
Source: EC2-2 Security Group

Never expose 9200 publicly.

⸻

28. ELASTICSEARCH TROUBLESHOOTING

Status:

sudo systemctl status elasticsearch

Logs:

sudo journalctl -u elasticsearch -n 100 –no-pager

Detailed Logs:

sudo tail -100 /var/log/elasticsearch/elasticsearch.log

⸻

29. KIBANA TROUBLESHOOTING

Status:

sudo systemctl status kibana

Logs:

sudo journalctl -u kibana -n 100 –no-pager

⸻

30. USEFUL ELASTIC COMMANDS

Generate Enrollment Token:

sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana

Reset Password:

sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic

Cluster Health:

curl -k -u elastic https://localhost:9200/_cluster/health?pretty

Indices:

curl -k -u elastic https://localhost:9200/_cat/indices?v

⸻

31. LESSONS LEARNED

1. Read logs first.
2. Test every layer independently.
3. Connectivity before configuration.
4. Elasticsearch 8.x uses HTTPS by default.
5. Filebeat modules require enabled filesets.
6. Verification Code and Enrollment Token are different.
7. Single-node and cluster settings must not be mixed.
8. Most failures are configuration issues.

⸻

32. FINAL OUTCOME

Successfully implemented:

EC2-1

* Elasticsearch
* Kibana
* Filebeat

EC2-2

* Filebeat

Result:

Centralized log collection from multiple Ubuntu servers into Elasticsearch with visualization through Kibana Discover.

Project Status:
SUCCESSFUL
