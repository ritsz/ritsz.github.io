---
title: Notes on Nginx
created: '2021-01-30T08:56:47.820Z'
modified: '2021-05-08T20:57:46.458Z'
---

#### NGINX is a web server
* A web server is a piece of software that responds to HTTP requests made by clients (usually web browsers). 
* The browser makes a request, and the web server responds with the static content (usually HTML) corresponding to that request.

#### NGINX is a reverse proxy
* A reverse proxy is a server that sits in front of a group of web servers. 
* When a browser makes an HTTP request, the request first goes to the reverse proxy, which then sends the request to the appropriate web server.

* Some benefits of a reverse proxy:
  * `Security`: with a reverse proxy, the web server never reveals its IP address to the client, which makes the server more secure.
  * `SSL Encryption (more security)`: encrypting and decrypting SSL communications is expensive, and would make web servers slow. The reverse proxy can be configured to decrypt incoming requests from the client and encrypt outgoing responses from the server.
  * `Load Balancing`: if a website is very popular, it’s unlikely that all the traffic is handled by a single web server. Usually, the website will be distributed across many web servers, with a reverse proxy in front of them. (see below for more on load balancing)

#### NGINX is a HTTP load balancer
* A load balancer is responsible for routing client HTTP requests to web servers in an efficient manner. This prevents any individual web server from being overworked.
* A load balancer can also be configured so that if a web server goes down, the reverse proxy will no longer forward requests to that web server

#### Nginx conf
* The location `/` means that when we visit the root url (`localhost:80/`), we use this configuration

```sh
$ cat /etc/nginx/nginx.conf

  user  nginx;
  worker_processes  1
  ...
  http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;
      ....
      access_log  /var/log/nginx/access.log  main;
      ...
      include /etc/nginx/conf.d/*.conf;   # <= Includes more conf files
  }

$ cat /etc/nginx/conf.d/default.conf
  server {
      listen       80;
      listen  [::]:80;
      server_name  localhost;			# <= What name and port the server is listening on

      location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
      }
  }
```

#### Nginx as reverse proxy
* Assume we have 2 nodejs web servers running on `localhost:5000` and `locahost:8000`, and we want to map them to `localhost:80/server1` and `localhost:80/server2`

```sh
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        server_name nodeserver;

        location /server1 {
                proxy_http_version 1.1;
                proxy_cache_bypass $http_upgrade;

                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_pass http://localhost:5000;
        }
        
        location /server2 {
                proxy_http_version 1.1;
                proxy_cache_bypass $http_upgrade;

                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_pass http://localhost:8000;
        }
}
```

#### Example: Nodejs server with a Nginx reverse proxy
* Code layout

```sh
[rritesh-a02:rritesh:~/Docker/nginx_revproxy_example] $ ls -lR
total 8
-rw-r--r--  1 rritesh  staff  245 Jan 30 15:32 docker-compose.yml  	# <= YAML file used by docker compose to start all the services
drwxr-xr-x  4 rritesh  staff  128 Jan 30 15:36 nginx 				        # <= Nginx project folder 
drwxr-xr-x  4 rritesh  staff  128 Jan 30 11:03 nodejs 				      # <= nodejs project folder

./nginx:
total 16
-rw-r--r--  1 rritesh  staff   60 Jan 30 15:31 Dockerfile
-rw-r--r--  1 rritesh  staff  911 Jan 30 15:36 default.conf

./nodejs:
total 16
-rw-r--r--  1 rritesh  staff    97 Jan 30 18:15 Dockerfile
-rw-r--r--  1 rritesh  staff   139 Jan 30 18:15 package.json
-rw-r--r--  1 rritesh  staff  1037 Jan 30 18:31 server.js
```
* Docker compose file for both `nodejs` and `nginx`. Created containers for both the type using the `Dockerfile` in the two directories.

```yaml
docker-compose.yml:
  version: "3.8"
  services:
      nodeserver:
          build:
              context: ./nodejs 		# <= Which folder has nodejs Dockerfile (Build context)
          ports:
              - "1234:3000" 			  # <= Port mappings from HOST to Container
      nginx:
          restart: always
          build:
              context: ./nginx 		  # <= Which folder has nginx Dockerfile
          ports:
              - "80:80" 				    # <= Starting with - means an ARRAY in YAML.
      redis-server:
          image: 'redis'
```
##### Nodejs
* `nodejs` server file and Docker file

```json
package.json
{
    "dependencies": {
        "express": "*",
        "redis": "2.8.0"
    },
    "scripts": {
        "start": "node server.js"
    }
}
```
```js
//server.js:
const express = require('express');
const redis = require('redis');

const app = express();
const client = redis.createClient({
    host: 'redis-server',
    port: 6379,
});

// Two get requests endpoints, have 2 counters
client.set('rootvisits', 0);
client.set('server1visits', 0);

app.get('/', (req, res) => {
    client.get('rootvisits', (err, visits) => {
        console.log("Root visit number: ", visits)
        res.send('Number of root visits ' + visits);
        client.set('rootvisits', parseInt(visits) + 1);
    });
});

app.get('/server1', (req, res) => {
    client.get('server1visits', (err, visits) => {
        console.log("Server1 visit number: ", visits)
        res.send('Number of server1 visits ' + visits);
        client.set('server1visits', parseInt(visits) + 1);
    });
});

app.listen(3000, () => {
    console.log('listening on port 3000');
});
```
```dockerfile
FROM node:alpine
WORKDIR '/app'
COPY package.json .
RUN npm install
COPY . .
CMD ["npm","start"]
```

##### Nginx

```
default.conf:
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name localhost;

    location / {
      proxy_http_version 1.1;
      proxy_cache_bypass $http_upgrade;

      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection 'upgrade';
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_pass http://nodeserver:3000;   <= The destination has the name of the nodejs service in the YAML file.
                            <= Note, the port here is 3000, NOT 1234. 1234 of HOST is mapped to 3000 of nodeserver
                            <= Nginx should redirect directly to nodeserver:3000, nginx and nodeserver are essentially on the SAME network, and can communicate to each other using service name.
    }
}
```
```dockerfile
FROM nginx
COPY default.conf /etc/nginx/conf.d/default.conf
```
* The networking between the service (check the IP Address)

```sh
$ docker ps -a
CONTAINER ID   IMAGE                               COMMAND                  CREATED          STATUS          PORTS                    NAMES
d67b3b2600db   nginx_revproxy_example_nodeserver   "docker-entrypoint.s…"   11 minutes ago   Up 11 minutes   0.0.0.0:1234->3000/tcp   nginx_revproxy_example_nodeserver_1
f5d394540fc2   nginx_revproxy_example_nginx        "/docker-entrypoint.…"   11 minutes ago   Up 11 minutes   0.0.0.0:8080->80/tcp     nginx_revproxy_example_nginx_1
39afbcd85082   redis                               "docker-entrypoint.s…"   11 minutes ago   Up 11 minutes   6379/tcp                 nginx_revproxy_example_redis-server_1

[rritesh-a02:rritesh:~/Docker/nginx_revproxy_example] $ docker exec -it d67b3b2600db ip addr show dev eth0
51: eth0@if52: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
        valid_lft forever preferred_lft forever

[rritesh-a02:rritesh:~/Docker/nginx_revproxy_example] $ docker exec -it f5d394540fc2 hostname -I
172.18.0.4 

[rritesh-a02:rritesh:~/Docker/nginx_revproxy_example] $ docker exec -it 39afbcd85082 hostname -I
172.18.0.2
```
