```
docker run -d --name expressDB -p 1521:1521 -p 5500:5500 --network=SOANet -e ORACLE_PWD=powerOfDb32 \
-v "$(pwd)/docker/database/oradata:/opt/oracle/oradata" container-registry.oracle.com/database/express:latest

docker run -it --name sqlcl --network=SOANet container-registry.oracle.com/database/sqlcl:latest sys/powerOfDb32@expressDB:1521/XE as sysdba


docker exec -it sqlcl sql sys/powerOfDb32@expressDB:1521/XEPDB1 as sysdba

docker run -i -t --name soaas --network=SOANet -p 7001:7001 -p 8001:8001 -v "$(pwd)/docker/soa-suite/data:/u01/oracle/user_projects" --env-file ./docker/soa-suite/adminserver.env.list --env-file ./docker/soa-suite/soaserver.env.list soa-suite
```