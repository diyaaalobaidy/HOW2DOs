# How to expose MongoDB to use in Tableau
- Tableau does not support MongoDB as a data source, to work around this issue, we need to convert the interaction with MongoDB to be as SQL queries. The key for this to succeed is to install the MongoDB Connector for BI
- The connector is available [here](https://www.mongodb.com/docs/bi-connector/current/tutorial/install-bi-connector-debian/) for Debian based OS
- Download and install the connector as directed in the link above
- We need to make the connector to be run as a system service, to do this, we need to make a configuration file, this config file is best to be stored at `/etc` directory
- Create certificates for the mongodb
- Create a script file
```bash
nano mongodb-ssl-setup.sh
```
- Write these commands
```bash
# https://www.mongodb.com/docs/bi-connector/v2.10/tutorial/ssl-setup/
mkdir /etc/ssl
cd /etc/ssl
echo "[req]"> openssl.conf
echo "distinguished_name = req_distinguished_name" >> openssl.conf
echo "req_extensions = v3_req" >> openssl.conf

echo "[req_distinguished_name]" >> openssl.conf
echo "commonName = qafdev.com" >> openssl.conf

echo "[v3_req]" >> openssl.conf
echo "subjectAltName = @alt_names" >> openssl.conf

echo "[alt_names]" >> openssl.conf
echo "DNS.1 = yourhostname" >> openssl.conf
echo "IP.1 = 127.0.0.1" >> openssl.conf

openssl genrsa -out /etc/ssl/mdbprivate.key -aes256 -passout pass:password
openssl req -x509 -new -key /etc/ssl/mdbprivate.key -days 3650 -out /etc/ssl/mdbca.crt -passin pass:password
openssl req -new -nodes -newkey rsa:2048 -keyout /etc/ssl/mdb.key -out /etc/ssl/mdb.csr
openssl x509 -CA /etc/ssl/mdbca.crt -CAkey /etc/ssl/mdbprivate.key -CAcreateserial -req -days 3650 -in /etc/ssl/mdb.csr -out /etc/ssl/mdb.crt -passin pass:password
cat /etc/ssl/mdb.key /etc/ssl/mdb.crt > /etc/ssl/mdb.pem
cd
```
Execute the file
```bash
sudo bash mongodb-ssl-setup.sh
```
- Edit mongodb configuration
```bash
nano /etc/mongod.conf
```
- Wrute the following configurations
```bash
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
#  dbPath: /hdd/a2/mongodb_database_critical
  journal:
    enabled: true
#  engine:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 20

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
  logRotate: reopen
# network interfaces
net:
  port: 30250
  bindIp: "127.0.0.1,10.0.0.2,10.8.0.1"
  ssl:
    mode: allowSSL

    PEMKeyPassword: password
    PEMKeyFile: '/etc/ssl/mdb.pem'
    CAFile: '/etc/ssl/mdbca.crt'
    clusterFile: '/etc/ssl/mdb.pem'
#    PEMKeyFile: /etc/ssl/mongodb/certificate.pem
#    CAFile: /etc/ssl/mongodb/ca.pem
#    PEMKeyPassword: password
    allowInvalidCertificates: true
    allowInvalidHostnames: true

#  tls:
#      mode: allowTLS
#      certificateKeyFile: /etc/ssl/mongodb/certificate.crt
#      allowConnectionsWithoutCertificates: true

# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: enabled

#operationProfiling:

#replication:
#  replSetName: "rs0"

#sharding:

## Enterprise-Only Options:

#auditLog:

#snmp:
setParameter:
  internalQueryMaxBlockingSortMemoryUsageBytes: 2097152000
  internalDocumentSourceLookupCacheSizeBytes: 2097152000
  internalQueryFacetBufferSizeBytes: 2097152000
  internalQueryMaxJsEmitBytes: 2097152000
  internalQueryMaxPushBytes: 2097152000
  internalQueryMaxRangeBytes: 2097152000
```
- Create a script file
```bash
nano mongosql-ssl-setup.sh
```
- Write the following commands
```bash
# https://www.mongodb.com/docs/bi-connector/v2.10/tutorial/ssl-setup/
mkdir /etc/ssl
cd /etc/ssl
echo "[req]"> openssl.conf
echo "distinguished_name = req_distinguished_name" >> openssl.conf
echo "req_extensions = v3_req" >> openssl.conf

echo "[req_distinguished_name]" >> openssl.conf
echo "commonName = qafdev.com" >> openssl.conf

echo "[v3_req]" >> openssl.conf
echo "subjectAltName = @alt_names" >> openssl.conf

echo "[alt_names]" >> openssl.conf
echo "DNS.1 = yourhostname" >> openssl.conf
echo "IP.1 = 127.0.0.1" >> openssl.conf

openssl req -new -nodes -newkey rsa:2048 -keyout /etc/ssl/bi.key -out /etc/ssl/bi.csr
openssl x509 -CA /etc/mongod/mdbca.crt -CAkey /etc/mongod/mdbprivate.key -CAcreateserial -req -days 1000 -in /etc/ssl/bi.csr -out /etc/ssl/bi.crt -passin pass:password
cat /etc/ssl/bi.key /etc/ssl/bi.crt > /etc/ssl/bi.pem
```
- Execute the file
```bash
sudo bash mongosql-ssl-setup.sh
```
- Create the configuration file using the following command
```bash
nano /etc/mongosql.conf
```
- Write the following configurations in the file
```bash
# MongoDB BI connector configuration
# Logging
systemLog:
  logAppend: false
  path: "/var/log/mongosqld.log"
  verbosity: 2

# Enable security to log in using a user and password
security:
  enabled: true

# Connect to MongoDB using a user and password
mongodb:
  net:
    uri: "127.0.0.1:30250" # Mongodb server is running on port 30250
    auth:
      username: "mongoUser"
      password: "godfather4mongo"
    ssl:
      enabled: true
      PEMKeyFile: '/etc/ssl/mdb.pem'
      CAFile: '/etc/ssl/mdbca.crt'
      allowInvalidCertificates: true
# Expose MongoDB BI connector as a service through port 34591
net:
  bindIp: 0.0.0.0 # make the service accessible from any IP
  port: 34591 # make the service listen on port 34591
  ssl: # enable SSL
    mode: requireSSL
    allowInvalidCertificates: true
    PEMKeyFile: '/etc/ssl/bi.pem'
    CAFile: '/etc/ssl/mdbca.crt'
``````
- Restart MongoDB
```bash
sudo service mongod restart
```
- Check the status of MongoDB
```bash
sudo service mongod status
```
- Restart MongoDB BI connector
```bash
sudo service mongosqld restart
```
- Check the status of MongoDB BI connector
```bash
sudo service mongosqld status
```
- Connect to MongoDB BI connector using mysql, and configure it in Tableau

## Problems
- I am having a problem connecting to the MongoSQL connector from MySQL, it is an ssl issue, I couldn't solve it yet
