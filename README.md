# Running SOA suite 12c in container

- got to https://container-registry.oracle.com/
- choose products you are interested and agree on terms and conditions

## Running multi-container Docker applications

>TODO docker-compose one the way :P

## Running Separate containers

### **Database Container**
- pull database image

>NOTE: I have used latest express edition database
```
docker pull container-registry.oracle.com/database/express:latest
```

Optional:
- create a folder where all the data files from db will be stored and accessible from host file system
```
mkdir docker/database/oradata
chmod 777 $(pwd)/docker/database/oradata
```

- create network for containers

```
docker network create -d bridge SOANet
```

- startup the database

```
docker run -d --name {containerName} --network={networkName} -p {dbPort}:1521 -p {emPort}:5500 -e ORACLE_PWD={dbPassword} container-registry.oracle.com/database/express:latest
```

> -v "$(pwd)/docker/database/oradata:/opt/oracle/oradata" is optional

| Param | Description |
| ----------- | ----------- |
| {containerName} | Name for docker container |
| {dbPort} | Port where on host machine DB will be accessible like localhost:1521 |
| {emPort} | Port where on host machine DB Enterprise manager will be ecessible https://localhost:5500/em |
| {dbPassword} | password for SYS |
| {networkName} | docker network name |

*Example:*
```
docker run -d --name expressDB --network=SOANet -p 1521:1521 -p 5500:5500 -e ORACLE_PWD={password}  -v "$(pwd)/docker/database/oradata:/opt/oracle/oradata" container-registry.oracle.com/database/express:latest
```

>Additional information could be found: https://container-registry.oracle.com/ords/f?p=113:4:100520576704568:::4:P4_REPOSITORY,AI_REPOSITORY,AI_REPOSITORY_NAME,P4_REPOSITORY_NAME,P4_EULA_ID,P4_BUSINESS_AREA_ID:803,803,Oracle%20Database%20Express%20Edition,Oracle%20Database%20Express%20Edition,1,0&cs=3AJ8KI5W3y0ZpNZERZfvQqbHhT2nhzozcf3VudKFPgHSYNOEcyZpqQKytmsy4T4fc9i0cJasRxJelWaovwtanwA

### **Running SQLCl check if database is accessible**

In order to verify if the database is acccepting connections from same container network and everything is working fine you can run the SQL Command Line interface towards the container and check connection.

Also please note that databse is accessible from host machine using localhost and port specified as {dbPort}.
ADDRESS = (PROTOCOL = TCP)(HOST = 127.0.0.1)(PORT = {dbPort})(SID = XE))


```
docker run -it --name sqlcl --network={networkName} container-registry.oracle.com/database/sqlcl:latest {dbUser}/{dbPassword}@{dbContainerName}:{dbPort}/{sid} as sysdba
```

| Param | Description |
| ----------- | ----------- |
| {dbContainerName} | container name of database |
| {dbPort} | database port |
| {dbPassword} | password for database user |
| {networkName} | docker network name |
| {sid} | sid of database, XE for CDB and XEPDB1 for pluggable |
| {dbUser} | database user |

*Example:*
```
docker run -it --name sqlcl --network=SOANet container-registry.oracle.com/database/sqlcl:latest sys/{passoword}@expressDB:1521/XE as sysdba
```
or
```
docker run -it --name sqlcl --network=SOANet container-registry.oracle.com/database/sqlcl:latest sys/{password}@expressDB:1521/XEPDB1 as sysdba
```
on running sqlcl container:
```
docker exec -it sqlcl sql sys/{password}@expressDB:1521/XEPDB1
```

### SOA Suite 
- pull database image

```
docker pull container-registry.oracle.com/middleware/soasuite:12.2.1.3
```

- create a folder where all the data files from soa will be stored and accessible from host file system
```
mkdir docker/soa-suite/data
mkdir docker/soa-suite/logs
chmod 777 $(pwd)/docker/soa-suite/logs
chmod 777 $(pwd)/docker/soa-suite/data
```

- prepare config files for SOA database setup (adminserver.env.list) and SOA managed server (soaserver.env.list)

**adminserver.env.list**

```
CONNECTION_STRING={dbContainerName}:{dbPort}/{sid}
RCUPREFIX=SOA1
DB_PASSWORD={dbPassword}
DB_SCHEMA_PASSWORD={soaSchemaPassword}
ADMIN_PASSWORD={adminPassword}
MANAGED_SERVER={soaServerName}
DOMAIN_TYPE=soa
```
**soaserver.env.list**

```
MANAGED_SERVER={soaServerName}
DOMAIN_TYPE=soa
DOMAIN_NAME=soainfra
ADMIN_HOST={soaContainerName}
ADMIN_PORT=7001
ADMIN_PASSWORD={adminPassword}
```

| Param | Description |
| ----------- | ----------- |
| {adminPassword} | administrative enterprise manager and weblogic console password |
| {dbContainerName} | database cointainer name |
| {dbPort} | database port  |
| {dbPassword} | password for system user that will be used to create SOA schema entries |
| {soaSchemaPassword} | password that will be used in SOA schema passowrds |
| {sid} | sid of database, XE for CDB and XEPDB1 for pluggable |
| {soaContainerName} | name of soa-suite container that will be used when starting |

> Sample Config files are avaliable in repo

- start container
```
docker run -i -t --name {soaContainerName} --network={networkName} -p {adminPort}:7001 -p {managedServerPort}:8001 --env-file ./docker/soa-suite/adminserver.env.list --env-file ./docker/soa-suite/soaserver.env.list container-registry.oracle.com/middleware/soasuite:12.2.1.3
```

| Param | Description |
| ----------- | ----------- |
| {soaContainerName} | name of soa-suite container that will be used when starting |
| {networkName} | docker network name |
| {adminPort} | Port where administrative tools are accessible, https://localhost:{adminPort}/console and https://localhost:{adminPort}/em |
| {managedServerPort} | Port managed server is accessible |



*Example*
```
docker run -i -t --name soaas --network=SOANet -p 7001:7001 -p 8001:8001 -v "$(pwd)/docker/soa-suite/data:/u01/oracle/user_projects" --env-file ./docker/soa-suite/adminserver.env.list --env-file ./docker/soa-suite/soaserver.env.list soa-suite
```



>Additional information could be found: https://container-registry.oracle.com/ords/f?p=113:4:116748029503871:::4:P4_REPOSITORY,AI_REPOSITORY,AI_REPOSITORY_NAME,P4_REPOSITORY_NAME,P4_EULA_ID,P4_BUSINESS_AREA_ID:252,252,Oracle%20SOA%20Suite,Oracle%20SOA%20Suite,1,0&cs=3sMy1ZkPPrdW8-wgHpKNjSTxlwYgdlsVnqu1IHNx7x3EY--YnLWOtaa42XsT4019CsYSPZ_NuAmQuBqNod8-sKQ


>Other usefull info: https://technology.amis.nl/soa/soa-suite-12c-in-docker-containers-only-a-couple-of-commands-no-installers-no-third-party-scripts/