# ERP-Next-V14-Production-Setup

`apt update`

`apt install nano`

`nano apps.txt`

```
frappe
payments
erpnext
```

`nano common_site_config.json`

```
{

  "db_host": "db", 
  "db_port": 3306, 
  "redis_cache": "redis://redis-cache:6379", 
  "redis_queue": "redis://redis-queue:6379", 
  "redis_socketio": "redis://redis-socketio:6379", 
  "socketio_port": 9000
  
}

```


# add new stack 
* `erpnext-v14`
* past the yml file 



```
version: "3.7"

services:
  backend:
    image: frappe/erpnext-worker:v14.0.3
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - assets:/home/frappe/frappe-bench/sites/assets
    networks:  
      - db-network
      - bench-network
    

  configurator:
    image: frappe/erpnext-worker:v14.0.3
    command:
      - configure.py
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      REDIS_SOCKETIO: redis-socketio:6379
      SOCKETIO_PORT: "9000"
    volumes:
      - sites:/home/frappe/frappe-bench/sites

  create-site:
    image: frappe/erpnext-worker:v14.0.3
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - assets:/home/frappe/frappe-bench/sites/assets
    entrypoint:
      - bash
      - -c
    command:
      - >
        wait-for-it -t 120 db:3306;
        wait-for-it -t 120 redis-cache:6379;
        wait-for-it -t 120 redis-queue:6379;
        wait-for-it -t 120 redis-socketio:6379;
        export start=`date +%s`;
        until [[ -n `grep -hs ^ common_site_config.json | jq -r ".db_host // empty"` ]] && \
          [[ -n `grep -hs ^ common_site_config.json | jq -r ".redis_cache // empty"` ]] && \
          [[ -n `grep -hs ^ common_site_config.json | jq -r ".redis_queue // empty"` ]];
        do
          echo "Waiting for common_site_config.json to be created";
          sleep 5;
          if (( `date +%s`-start > 120 )); then
            echo "could not find common_site_config.json with required keys";
            exit 1
          fi
        done;
        echo "common_site_config.json found";
        bench new-site frontend --admin-password=admin --db-root-password=admin --install-app payments --install-app erpnext --set-default;
  db:
    image: mariadb:10.6
    healthcheck:
      test: mysqladmin ping -h localhost --password=admin
      interval: 1s
      retries: 15
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed # Temporary fix for MariaDB 10.6
    environment:
      MYSQL_ROOT_PASSWORD: admin
    volumes:
      - db-data:/var/lib/mysql
    networks:  
      - db-network  

  frontend:
    image: frappe/erpnext-nginx:v14.0.3
    deploy:
      restart_policy:
        condition: on-failure
      labels:
        traefik.docker.network: traefik-public
        traefik.enable: "true"
        traefik.constraint-label: traefik-public
        # Change router name prefix from erpnext- to the name of stack in case of multi bench setup
        traefik.http.routers.erpnext--http.rule: Host(${SITES:?No sites set})
        traefik.http.routers.erpnext--http.entrypoints: http
        # Remove following lines in case of local setup
        traefik.http.routers.erpnext--http.middlewares: https-redirect
        traefik.http.routers.erpnext--https.rule: Host(${SITES})
        traefik.http.routers.erpnext--https.entrypoints: https
        traefik.http.routers.erpnext--https.tls: "true"
        traefik.http.routers.erpnext--https.tls.certresolver: le
        # Remove above lines in case of local setup
        traefik.http.services.erpnext-.loadbalancer.server.port: "8080"  
    environment:
      BACKEND: backend:8000
      FRAPPE_SITE_NAME_HEADER: frontend
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: 127.0.0.1
      UPSTREAM_REAL_IP_HEADER: X-Forwarded-For
      UPSTREAM_REAL_IP_RECURSIVE: "off"
    volumes:
      - sites:/usr/share/nginx/html/sites
      - assets:/usr/share/nginx/html/assets
    networks:
      - db-network
      - bench-network
      - traefik-public  


  queue-default:
    image: frappe/erpnext-worker:v14.0.3
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - worker
      - --queue
      - default
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:  
      - db-network
      - bench-network  

  queue-long:
    image: frappe/erpnext-worker:v14.0.3
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - worker
      - --queue
      - long
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:  
      - db-network
      - bench-network    

  queue-short:
    image: frappe/erpnext-worker:v14.0.3
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - worker
      - --queue
      - short
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:  
      - db-network
      - bench-network    

  redis-queue:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - redis-queue-data:/data
    networks:  
      - bench-network    

  redis-cache:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - redis-cache-data:/data
    networks:  
      - bench-network  

  redis-socketio:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - redis-socketio-data:/data
    networks:  
      - bench-network  

  scheduler:
    image: frappe/erpnext-worker:v14.0.3
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - schedule
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:  
      - db-network
      - bench-network    

  websocket:
    image: frappe/frappe-socketio:v14.5.0
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:  
      - db-network
      - bench-network    
      

volumes:
  assets:
  db-data:
  redis-queue-data:
  redis-cache-data:
  redis-socketio-data:
  sites:
  
networks:
  bench-network:
    # Change network name from erpnext-v13 to the name of stack in case of multi bench setup
    name: erpnext-v14
    external: false
  db-network:
    name: db-network
    external: false
  traefik-public:
    name: traefik-public
    external: true
  
```

# Add ENV

* `SITES` = `dev-erp14.arcapps.org`
