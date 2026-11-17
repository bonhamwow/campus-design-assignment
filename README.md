# iptables-firewall-simulation-lab
## Obective:
In this lab, I create a network design for a single campus simulation using Containerlab, Alpine Linux switches, iptables, and OWASP ModSecurity WAF.

It is a 4-subnet campus environment connecting Students, Faculty, IT Admin staff and Centralized Serices. The network requires Layer 3/4 Packet Filtering Firewalls on the central router and Layer 7 Web Application Firewall to defend against web exploits. 

The network meets the following topology:
```mermaid
flowchart TD
    subgraph STUDENT ["Student Subnet (10.10.10.0/24)"]
        student_node["Student Client<br/>10.10.10.10/24<br/>GW: 10.10.10.1"]
    end

    subgraph FACULTY ["Faculty Subnet (10.10.20.0/24)"]
        faculty_node["Faculty Client<br/>10.10.20.10/24<br/>GW: 10.10.20.1"]
    end

    subgraph ADMIN ["IT Admin Subnet (10.10.30.0/24)"]
        admin_node["IT Admin Client<br/>10.10.30.10/24<br/>GW: 10.10.30.1"]
    end

    subgraph GATEWAY ["Central Perimeter Router / Firewall"]
        router["Alpine Router (iptables)<br/><br/>eth1: 10.10.10.1/24 (Student GW)<br/>eth2: 10.10.20.1/24 (Faculty GW)<br/>eth3: 10.10.30.1/24 (Admin GW)<br/>eth4: 10.10.40.1/24 (Services GW)"]
    end

    subgraph SERVICES ["Services Subnet (10.10.40.0/24)"]
        direction TB
        waf["ModSecurity WAF Proxy<br/>10.10.40.80:8080<br/>GW: 10.10.40.1"]
        juiceshop["OWASP Juice Shop<br/>(Bulletin App)<br/>10.10.40.81:3000<br/>GW: 10.10.40.1"]
        postgres["PostgreSQL Database<br/>(Grading DB)<br/>10.10.40.100:5432<br/>GW: 10.10.40.1"]
    end

    student_node ---|eth1 <--> eth1| sw_student["Alpine Switch (sw-student)"]
    sw_student ---|eth2 <--> eth1| router
    
    faculty_node ---|eth1 <--> eth1| sw_faculty["Alpine Switch (sw-faculty)"]
    sw_faculty ---|eth2 <--> eth2| router
    
    admin_node ---|eth1 <--> eth1| sw_admin["Alpine Switch (sw-admin)"]
    sw_admin ---|eth2 <--> eth3| router
    
    waf ---|eth1 <--> eth1| sw_services["Alpine Switch (sw-services)"]
    juiceshop ---|eth1 <--> eth2| sw_services
    postgres ---|eth1 <--> eth3| sw_services
    sw_services ---|eth4 <--> eth4| router

    classDef admin fill:#ffe6e6,stroke:#d9534f,stroke-width:2px;
    classDef faculty fill:#fff0e6,stroke:#f0ad4e,stroke-width:1px;
    classDef student fill:#e6ffe6,stroke:#5cb85c,stroke-width:1px;
    classDef services fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef router fill:#f9f,stroke:#333,stroke-width:2px;

    class ADMIN admin;
    class FACULTY faculty;
    class STUDENT student;
    class SERVICES services;
    class GATEWAY router;
```
---
## Requirements to Demo this Lab
This lab was created inside Containerlabs and is meant to be run within Linux or WSL. The YAML file and config files are available from the repository.

# Running the Lab
## Building the Custom Images
This lab makes use of two custom Docker images. The following commands must be entered before deploying the lab:
```
docker build -t custom-juiceshop:latest ./images/juiceshop
```
```
docker build -t custom-waf:latest ./images/modsec
```

## Deploying the Lab
Open Containerlabs and navigate to the downloaded YAML file, then run the following command:
```
sudo containerlab deploy -t lab.clab.yml
```
## Testing the Firewall
To test connections between subnets on the network, run the following commands:

### **Test 1: Legitimate Bulletin Web Access (Student to WAF)**
* **Command:** `docker exec -it clab-campus-network-security-student-client curl -s -o /dev/null -w "%{http_code}" http://10.10.40.80:8080/`
* **Expected Result:** `200`

### **Test 2: ModSecurity WAF Interception of SQL Injection**
* **Command:** `docker exec -it clab-campus-network-security-student-client curl -s -o /dev/null -w "%{http_code}" "http://10.10.40.80:8080/rest/user/login?email='OR'1'='1"`
* **Expected Result:** `403`

### **Test 3: Authorized Grading DB Access (Faculty Subnet)**
* **Command:** `docker exec -it clab-campus-network-security-faculty-client nc -zv -w 3 10.10.40.100 5432`
* **Expected Result:** `open` / connection succeeded

### **Test 4: Unauthorized Grading DB Access Block (Student Subnet)**
* **Command:** `docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.100 5432`
* **Expected Result:** Connection timeout / failure

### **Test 5: SSH Privilege Validation (IT Admin Host Allowed)**
* **Command:** `docker exec -it clab-campus-network-security-admin-client nc -zv -w 3 10.10.40.100 22`
* **Expected Result:** Connection refused (packet reaches target, confirming rule allows traversal, but port 22 is unlistened on DB)

### **Test 6: SSH Privilege Restriction (Student Host Blocked)**
* **Command:** `docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.100 22`
* **Expected Result:** Connection timeout / failure (dropped by router firewall)
---
## Destroying the Lab
Once you've finished with the lab, run the following command to destroy and clean-up:
```
sudo containerlab destroy -t lab.clab.yml
```
