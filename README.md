# lift-shift-migration-guide
Lift and shift migration—also known as rehosting—is a cloud migration strategy that moves applications and their data from an on‑premises environment to a cloud infrastructure without significant changes to their architecture or code. This approach emphasizes speed and minimal disruption, allowing organizations to begin leveraging cloud scalability and cost models quickly, though it may limit access to deeper cloud‑native optimizations and services.

# What Is Lift and Shift Migration?
Lift and shift migration involves taking an existing application “as is” and redeploying it onto cloud‑based virtual machines or infrastructure‑as‑a‑service (IaaS) resources, without altering its core design or dependencies.It’s often the first step organizations use to “park” workloads in the cloud while planning longer‑term modernization or refactoring efforts.

# Lift and Shift Migration of Java Application on Azure
Here we describe how you can perform a lift and shift migration of a Java web application with a MariaDB backend onto Azure virtual machines (VMs). The architecture consists of:

VM1: MariaDB (MySQL-compatible) database
VM2: Tomcat application server hosting the Java application
VM3: Memcached caching layer
VM4: RabbitMQ messaging broker

Each VM is provisioned via Azure CLI using a cloud-init (user‑data) script for automated OS setup, package installation, and service configuration.

# 1. Architecture Diagram
![Architecture-RG](https://github.com/user-attachments/assets/3851b233-f5f1-4368-803e-082e00a31f6d)
Figure: Four-VM topology within a single Azure Virtual Network (db01-vnet), with NSGs and Public IPs.

# 2. Prerequisites
  1. Azure CLI installed and logged in (az login).
  2. A Resource Group for all resources:
       az group create \
    --name rg-liftshift \
    --location eastus
  3. A Virtual Network and subnet:
     az network vnet create \
    --resource-group rg-liftshift \
    --name db01-vnet \
   --address-prefix 10.0.0.0/16 \
   --subnet-name app-subnet \
   --subnet-prefix 10.0.1.0/24

# 3. Networking & Security Groups
Create one Network Security Group (NSG) per VM type, opening only required ports:

VM Role            NSG Name          Open Ports

Database           nsg-db            TCP 3306

Tomcat             nsg-tomcat        TCP 8080, 22

Memcached         nsg-memcache       TCP 11211

RabbitMQ          nsg-rabbitmq       TCP 5672, 15672, 22

Example for MariaDB NSG:
 az network nsg create \
  --resource-group rg-liftshift \
  --name nsg-db
az network nsg rule create \
  --resource-group rg-liftshift \
  --nsg-name nsg-db \
  --name allow-mysql \
  --protocol Tcp \
  --direction Inbound \
  --priority 100 \
  --source-address-prefixes '*' \
  --destination-port-ranges 3306 \
  --access Allow

Repeat similarly for the other roles, adjusting ports...


# 4. VM Provisioning with User‑Data (Cloud‑Init)
Below, we provision all four VMs with Azure CLI, passing a --custom-data file that contains the cloud‑init script. Create individual .yml scripts for each role.

# 4.1 VM1: MariaDB Server
cloud-init-mariadb.yml:
package_update: true
packages:
  - mariadb-server
runcmd:
  - systemctl enable mariadb
  - systemctl start mariadb
  - mysql -e "CREATE DATABASE appdb;"
  - mysql -e "CREATE USER 'appuser'@'%' IDENTIFIED BY 'ChangeMe123!';"
  - mysql -e "GRANT ALL ON appdb.* TO 'appuser'@'%';"
  - mysql -e "FLUSH PRIVILEGES;"

Provisioning command:
az vm create \
  --resource-group rg-liftshift \
  --name vm-mariadb \
  --image UbuntuLTS \
  --size Standard_B1ms \
  --vnet-name db01-vnet \
  --subnet app-subnet \
  --nsg nsg-db \
  --public-ip-address vm-mariadb-ip \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init-mariadb.yml

# 4.2. VM2: Tomcat Application Server
cloud-init-tomcat.yml:
package_update: true
packages:
  - default-jdk
  - tomcat10
runcmd:
  - systemctl enable tomcat10
  - systemctl start tomcat10
  - wget -O /var/lib/tomcat10/webapps/app.war "https://your.artifact.repo/app.war"

Provisioning:
az vm create \
  --resource-group rg-liftshift \
  --name vm-tomcat \
  --image UbuntuLTS \
  --size Standard_B1ms \
  --vnet-name db01-vnet \
  --subnet app-subnet \
  --nsg nsg-tomcat \
  --public-ip-address vm-tomcat-ip \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init-tomcat.yml


# 4.3. VM3: Memcached Server
cloud-init-memcache.yml:
package_update: true
packages:
  - memcached
runcmd:
  - systemctl enable memcached
  - systemctl start memcached

Provisioning:
az vm create \
  --resource-group rg-liftshift \
  --name vm-memcache \
  --image UbuntuLTS \
  --size Standard_B1ms \
  --vnet-name db01-vnet \
  --subnet app-subnet \
  --nsg nsg-memcache \
  --public-ip-address vm-memcache-ip \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init-memcache.yml


# 4.4. VM4: RabbitMQ Server
cloud-init-rabbitmq.yml:
package_update: true
packages:
  - rabbitmq-server
runcmd:
  - systemctl enable rabbitmq-server
  - systemctl start rabbitmq-server
  - rabbitmqctl add_user appuser ChangeMe123!
  - rabbitmqctl set_permissions -p / appuser ".*" ".*" ".*"

Provisioning:
az vm create \
  --resource-group rg-liftshift \
  --name vm-rabbitmq \
  --image UbuntuLTS \
  --size Standard_B1ms \
  --vnet-name db01-vnet \
  --subnet app-subnet \
  --nsg nsg-rabbitmq \
  --public-ip-address vm-rabbitmq-ip \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init-rabbitmq.yml

# 5. Application Configuration (application.properties)
Configure your Spring Boot (or similar) Java application to point to these services over the VNet:
# Database (MariaDB)
spring.datasource.url=jdbc:mysql://<private-ip-of-vm-mariadb>:3306/appdb
spring.datasource.username=appuser
spring.datasource.password=ChangeMe123!

# Memcached
spring.cache.type=memcached
memcached.servers=<private-ip-of-vm-memcache>:11211

# RabbitMQ
spring.rabbitmq.host=<private-ip-of-vm-rabbitmq>
spring.rabbitmq.port=5672
spring.rabbitmq.username=appuser
spring.rabbitmq.password=ChangeMe123!

Replace <private-ip-of-*> with the internal IPs assigned to each VM’s NIC.


# 6. Testing & Validation
1. SSH into each VM to confirm services are running:
     ssh azureuser@<public-ip-vm>
    systemctl status mariadb    # on VM1
    systemctl status tomcat9    # on VM2
    systemctl status memcached  # on VM3
    systemctl status rabbitmq-server  # on VM4

2. Application Smoke Test: Browse to http://<public-ip-vm-tomcat>:8080/app/health (or your endpoint) to verify connectivity to DB, cache, and queue.
3. Performance Check: Use simple load testing to verify expected throughput.
