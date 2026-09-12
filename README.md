# Scenario 1

Use one heading for each problem.
Write what was wrong and how you fixed it.
Paste the config you changed (only the changed part).
Paste the commands you used.
Write every step you tried, even guesses.

English is better. Persian is OK.

## Problem 1: Nginx Upstream Misconfiguration
What was wrong:
The nginx.conf file was configured to proxy traffic to http://backend-api:8080. However, the backend service is named backend (not backend-api), and according to the Dockerfile and entrypoint.sh, Gunicorn starts the backend application listening on port 5000 (not 8080). This caused a 502 Bad Gateway error because Nginx could not find the upstream host on the specified port.

How I fixed it:
I used the sed command to replace http://backend-api:8080 with http://backend:5000 in the /opt/service-catalog/nginx/nginx.conf file.

Config I changed (only the changed part):
```
# /opt/service-catalog/nginx/nginx.conf
# Changed from: set $backend_upstream http://backend-api:8080;
# Changed to:
set $backend_upstream http://backend:5000;
```

Commands I used:

```

cd /opt/service-catalog/
ls -la
ls -la /opt/service-catalog/nginx/
cat /opt/service-catalog/nginx/nginx.conf
cat /opt/service-catalog/backend/Dockerfile
cat /opt/service-catalog/backend/entrypoint.sh
cat /opt/service-catalog/backend/app.py

# Fix and verify:
sed -i 's|http://backend-api:8080|http://backend:5000|g' /opt/service-catalog/nginx/nginx.conf
grep backend_upstream /opt/service-catalog/nginx/nginx.conf
```
reslut:
```
root@reserve-5-scenario1:/opt/service-catalog/backend# curl -i http://localhost/graph
HTTP/1.1 200 OK
Server: nginx/1.27.5
Date: Sat, 12 Sep 2026 12:43:32 GMT
Content-Type: application/json
Content-Length: 743
Connection: keep-alive
{"edges":[{"from":"frontend","id":1,"to":"backend"},{"from":"frontend","id":2,"to":"redis"},{"from":"backend","id":3,"to":"postgres"},{"from":"backend","id":4,"to":"redis"},{"from":"backend","id":5,"to":"auth"},{"from":"worker","id":6,"to":"postgres"},{"from":"worker","id":7,"to":"kafka"},{"from":"auth","id":8,"to":"postgres"}],"nodes":[{"id":1,"kind":"service","name":"frontend","type":"frontend"},{"id":2,"kind":"service","name":"backend","type":"backend"},{"id":3,"kind":"service","name":"worker","type":"worker"},{"id":4,"kind":"service","name":"auth","type":"auth"},{"id":5,"kind":"infra","name":"postgres","type":"database"},{"id":6,"kind":"infra","name":"redis","type":"cache"},{"id":7,"kind":"infra","name":"kafka","type":"queue"}]}
```
## Problem 2:Docker Network Isolation in docker-compose.yml
What was wrong:
In the provided docker-compose.yml, the backend service was attached to nginx-backend-net and the db service was attached to backend-db-net. Because they were on separate, isolated Docker networks, the backend container could not resolve or connect to the db hostname, causing the application to crash with Connection refused when trying to reach PostgreSQL.

How I fixed it:
Since we could not use docker-compose, I created a single shared Docker network named app-net and manually ran the db, backend, and nginx containers attached to this same network so they could communicate freely.

Config I changed (only the changed part):
```
# /opt/service-catalog/docker-compose.yml
# (If we were using docker-compose, the fix would be to put backend and db on the same network)
  backend:
    networks:
      - nginx-backend-net
      - backend-db-net   # <-- Added this network to backend

  db:
    networks:
      - backend-db-net
```
Commands I used:
```
# Investigation steps:
cat /opt/service-catalog/docker-compose.yml
docker ps -a
docker logs backend

# Fix: Create a single network and start containers on it
docker network create app-net

docker run -d --name db --network app-net \
  -e POSTGRES_USER=catalog \
  -e POSTGRES_PASSWORD=catalog \
  -e POSTGRES_DB=catalog \
  postgres:16-alpine

docker run -d --name backend --network app-net \
  -e DATABASE_URL="postgresql+psycopg2://catalog:catalog@db:5432/catalog" \
  service-catalog:latest

docker run -d --name nginx --network app-net \
  -p 80:80 \
  -v /opt/service-catalog/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:1.27-alpine

# Verify:
sleep 5
docker ps -a
curl -i http://localhost/graph
```

Problem 3: Missing docker-compose plugin and unable to install
What was wrong:
The VM had an older Docker version that did not support the docker compose plugin, and the standalone docker-compose binary was not installed. We could not install it using apt because the VM had no internet access due to a DNS resolver problem.

How I fixed it:
I bypassed the need for docker-compose by checking locally cached Docker images (docker images) and running the containers manually with standard docker run commands on a manually created network.

Config I changed (only the changed part):
```
# No config file was changed for this. 
# We used standard docker CLI commands instead of docker-compose.
```
commands i uses:
```
# Investigation steps:
docker-compose version
apt update
apt install -y docker-compose
ping -c 2 google.com
ping -c 2 8.8.8.8
docker compose version
systemctl status docker
docker images

# Fix: Running containers manually (commands shown in Problem 2)
# Final test:
curl -i http://localhost/graph
```


# Extra problems

Write side problems here. For example: your laptop, a wrong config change, or internet.
Write how much time each one took.

DNS resolver has a problem and we could not download apt packages (15 min). The VM could ping IPs like 8.8.8.8 but could not resolve domain names like google.com or nova.clouds.archive.ubuntu.com, preventing us from installing docker-compose.
Race condition during testing (5 min). When running curl immediately after docker run, I got a 502 Bad Gateway because the backend was still waiting for PostgreSQL to boot up. I checked docker logs backend and saw it eventually connected, and the subsequent curl returned HTTP 200.

