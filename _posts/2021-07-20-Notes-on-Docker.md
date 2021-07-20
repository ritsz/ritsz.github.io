---
title: DOCKER
created: '2021-02-22T04:15:04.561Z'
modified: '2021-05-08T16:25:24.145Z'
---
* Docker is a platform or ecosystem around creating and running containers.
* Image: Single file with all the deps and configs required to run a program
	Images are read-only templates with instructions to create a docker container
	* File System Snapshot (Example: bin, dev, etc, home, proc, root files for Busybox. Other apps have different file system.)
	* Startup command
* Container: Instance of a running image that runs a program. A program with its own set of hardware resources.
	* Uses:
		* Namespaces : Resource Isolation
		* Cgroups : Resource limiting
		* Seccomp-bpf : Limiting syscalls
* Docker makes it really easy to install and run software without worrying about setup or dependencies.
	Redis: `docker run -it redis`
		* `docker-cli` (docker client) reaches docker-hub, downloads an image and run as a container.
* Docker client: Tool that commands are issued to
* Docker Server (daemon): Responsible for creating images, running containers, etc.
	* The Docker client contacted the Docker daemon.
		* The Docker daemon could not find the "hello-world" image in the Image Cache. 
	* The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
	* The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
	* The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
* When docker is installed on MacOS/Windows, the Docker Server is run as a Linux Virtual Machine.

#### Building an image
* Sample dockerfile
```
Dockerfile
	FROM node:12-slim
	COPY server.js server.js
	CMD ["node", "server.js"]
```
* Each step  `FROM`, `COPY`, `CMD` creates an image that is consumed by the subsequent step. A `NodeJS` Image is created, a temporary container is created where a file is copied. This new container is used to generate a snapshot (image-2). This image-2 is then used to create a final image which has the Startup command set as `CMD`.
* Javascript code
```js
server.js
	const http = require("http");

	const app = (req, res) => {
	  console.log("ping!");
	  res.end("Hello there.", "utf-8");
	}

	http.createServer(app).listen(3000);
	console.log("server started");
```
* Building the image
```sh
docker build -t rritesh-node-img .
```
* Downloads node baseimage from Docker-Hub and creates the template image, by Copying the `server.js`. Doesn't run the `CMD`. 
* Tags the images as `rritesh-node-img`. The `image` command can be used to check the images. 
```sh
docker image ls
REPOSITORY         TAG       IMAGE ID       CREATED         SIZE
rritesh-node-img   latest    bc63aa5603f8   7 minutes ago   142MB
hello-world        latest    bf756fb1ae65   13 months ago   13.3kB

docker create --name my-app --init -p 3000:3000 rritesh-node-img 			
	703e2532312a6ace890b97e2bdcd88c03f19435b5f952015c373831df72dd0ae
```
* `-p` does port mapping between localhost and docker container.
* `--init` uses Tini package that handles `Ctrl-C`

#### Playing around with docker
```sh
docker ps -a
CONTAINER ID   IMAGE              COMMAND                  CREATED          STATUS                      PORTS     NAMES
703e2532312a   rritesh-node-img   "docker-entrypoint.s…"   24 seconds ago   Created                               my-app

docker start my-app
my-app

docker ps -a
CONTAINER ID   IMAGE              COMMAND                  CREATED             STATUS                         PORTS                    NAMES
703e2532312a   rritesh-node-img   "docker-entrypoint.s…"   2 minutes ago       Up 5 seconds                   0.0.0.0:3000->3000/tcp  my-app

docker attach my-app			# Attaches to the docker container and prints the logs; Ctrl-C to exit, which also stops the app
ping!
ping!
ping!
```
* `attach` command attaches to the primary process of the container, PID 1.
* If you run `attach`, your terminal is going to get stuck displaying the container logs. Worse, if you `attach` and then `ctrl-C` to get out, it will actually stop the container on exit. Or worse, it will just ignore the ctrl-C and trap your terminal. If that ever happens to you, you’ll have to open a new terminal and stop the container. That’s why in your applications you should handle the `SIGTERM` or use the Tini package like we have in our example. A better way to see output is the Docker logs command:
```sh
docker logs -f my-app
server started
ping!
ping!
ping!
```
* Run a bash shell on the container, notice that the server.js file is copied on the root mount point
```sh
docker exec -it my-app bash
root@703e2532312a:/# ls 			# <= root@<docker ID>
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  server.js  srv  sys  tmp  usr  var
```
* Stop the container: Sends a `SIGTERM` to the primary process in the container
```
$docker stop my-app
my-app

docker ps -a
CONTAINER ID   IMAGE              COMMAND                  CREATED             STATUS                         PORTS     NAMES
703e2532312a   rritesh-node-img   "docker-entrypoint.s…"   13 minutes ago      Exited (143) 3 seconds ago               my-app
```
* `docker kill` can be used to send SIGKILL. If docker stop is not handled by container in 10 sec, a docker kill is sent to it.
* Remove container
```sh	
docker rm my-app
	my-app
	
docker ps -a
	CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
* `run` is a shortcut that takes care of create, start, and attach all at once. `--rm` flag removes the container when we stop it (stop+rm).
```sh
docker run -d --name my-app -p 3000:3000 --init --rm rritesh-node-img
	700e9387cbca4e7d95f456136bd0046566471bc2130d48cdfe34241ea7ded05c
	
docker ps -a
	CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                    NAMES
	700e9387cbca   rritesh-node-img   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds   0.0.0.0:3000->3000/tcp   my-app
```
* `docker system prune` will remove all stopped containers, dangling images and build cache.
* `-i` => interactive: The STDOUT of terminal is piped to the STDIN of the running container, and vice versa
* `-t` => psuedo-tty: Give a psuedo terminal to interact with.
```sh
[rritesh-a02:rritesh:~] docker exec -i fd0c9599579f redis-cli (# Notice no psuedo terminal)
	set mynumber "5"
	OK
	^C

[rritesh-a02:rritesh:~] docker exec -it fd0c9599579f redis-cli
	127.0.0.1:6379> get mynumber
	"5"
```
* `docker commit` is used for converting a container into an image.
* `docker-compose up -d` => Create multiple containers and network them (-d means detach). Refer Nginx notes
* `docker-compose down` => Brings down (stops) all the containers in the current compose.

#### Mapping volumes
* Changing `index.html` file everytime and rebuilding the nginx image will take too much time.
* Instead a local volume of the HOST can be mapped to the container such that the `index.html` is taken from there:
```sh
docker run -d --name nginx-app -p 8000:80 -v /tmp/nginx/html:/usr/share/nginx/html:ro --init --rm nginx
```

#### Multi step build
* Outputs from one image build can be taken as the input to another image build
```
		FROM node:alpine as builder
		WORKDIR '/app'
		COPY package.json .
		RUN npm install
		COPY . .
		RUN npm run build
		 
		FROM nginx
		COPY --from=builder /app/build /usr/share/nginx/html
```
* The output of node app build `/app/build` is taken and copied over to nginx image. This final image will have an nginx server serving `node.js` web service.


* To make local docker daemon point to the docker registry inside minikube
```sh
> minikube docker-env
export DOCKER_TLS_VERIFY=”1"
export DOCKER_HOST=”tcp://172.17.0.2:2376"
export DOCKER_CERT_PATH=”/home/user/.minikube/certs”
export MINIKUBE_ACTIVE_DOCKERD=”minikube”
# To point your shell to minikube’s docker-daemon, run:
# eval $(minikube -p minikube docker-env)

> eval $(minikube -p minikube docker-env)
``` 
