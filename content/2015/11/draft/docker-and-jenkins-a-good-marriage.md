+++
date = "2015-11-10T14:04:48+01:00"
draft = true
title = "Docker & Jenkins... A good marriage"

+++

# Jenkins and Docker... A Good Marriage

We will run Jenkins and his slaves inside Docker container.

Moreover, Jenkins job will specify a docker images in which it need to be run.

With that way, we will have a fully containerized Jenkins eco-system, reliable, scalable and independent of the host environment.

## Jenkins - Master, Data and Nginx

For the Jenkins Master part, we will have 3 containers:

1. Jenkins Master

  Will handle the jenkins process.
2. Jenkins data

  JENKINS_HOME will be store inside this container.
3. Nginx

  We will use Nginx as Reverse Proxy.

Create a file `docker-compose.yml` with:
```yaml
# It's a Docker Data Volume, used to make data persistent containers
# Will handle the JENKINS_HOME data
data:
  image: docker-registry:5000/d.incubator/jenkins-data

# Will handle Jenkins process
# We are mounting volumes from jenkinsdata container
# So, it will persist his data to the jenkinsdata container
process:
  image: docker-registry:5000/d.incubator/jenkins-master
  volumes_from:
    - jenkinsdata

# Nginx used as Reverse Proxy
# Redirect port 80 of host to jenkins 8080 port
# We are linking nginx and jenkins-master containers
# With link, containers works as they are in the same network.
nginx:
  image: docker-registry:5000/d.incubator/jenkins-nginx
  ports:
    - "80:80"
  links:
    - process:jenkins-master
```

### Run them all, Folks

Now, we will use [docker-compose](Docker-Compose link), Docker-compose help you to manage run of multiple container.

Go to the folder where you create docker-compose.yml and run:

```bash
# Download the necessary images
> docker-compose pull
Pulling jenkinsdata (docker-registry:5000/d.incubator/jenkins-data:latest)...
latest: Pulling from d.incubator/jenkins-data
8a648f689ddb: Pull complete
2910de3f7bf9: Pull complete
eb3f8a9c09d8: Pull complete
0cb1e88f44bb: Pull complete
1b11166656fa: Pull complete
...

# Run docker-compose up -d to start all container
# -d is for running container as daemon

> docker-compose up -d
Creating jenkinsmaster_data_1
Creating jenkinsmaster_master_1
Creating jenkinsmaster_nginx_1

# Verify that container are running
> docker ps
CONTAINER ID  IMAGE                                           COMMAND                CREATED        STATUS              PORTS                         NAMES
c80f7f84fed5  docker-registry:5000/d.incubator/jenkins-nginx  "nginx -g 'daemon off" 31 seconds ago Up 30 seconds 443/tcp, 0.0.0.0:80->80/tcp   jenkinsmaster_master_1
79f4964a3752  docker-registry:5000/d.incubator/jenkins-master "/bin/tini -- /usr/lo" 31 seconds ago Up 30 seconds 8080/tcp, 50000/tcp           jenkinsmaster_nginx_1
```

You can see that the jenkinsmaster_data_1 container is not running.
Docker Data Volume is not running container. You can view it as a hard drive plug in the container.

You can test the Jenkins installation by accessing to http://<MACHINE-IP>, you should see the Jenkins home page.
![Jenkins Home Page](/images/2015/11/jenkins-home-page.png)

## Jenkins Slave - Swarm them and Docker-ception

We will use the Swarm plugin of Jenkins as Discovery Service.
The plugin is already installed on the master.
We need now to start a slave container that will connect to the master and be available to run job.

We will use a docker-compose file to make easy de container deployment

```yaml
swarmslave:
  build: jenkins-swarm-slave
  hostname: jenkins_swarm_slave_1
  environment:
    - JENKINS_SERVER=<IP-OF-DOCKER-MACHINE-WITH-JENKINS>
    - JENKINS_PORT=80
    - JENKINS_LABELS=swarm
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - <PATH-TO-DOCKER>/docker:/bin/docker
    - /tmp:/tmp
    - /usr/share/jenkins_slave:/usr/share/jenkins_slave

```

> Replace <IP-OF-DOCKER-MACHINE-WITH-JENKINS> with the Ip of your jenkins server ip

> Replace <PATH-TO-DOCKER> with the path to your docker client, you can find it with `which docker`

Than run your jenkins slave:

```bash
> docker-compose build

> docker-compose up -d
```

Now, if you go to http://<jenkins-url>/computer/, you should see your new slave:
![Jenkins computer page](/images/2015/11/jenkins-computer-page.png)

## Job configuration for using Jenkins Swarm

1. Create a *Freestyle Job*
2. Check *Restrict where this project can be run* and write `swarm` as *Label*
3. Deploy *Advanced Project Options*
4. Check *Use custom workspace* and write `/usr/share/jenkins_slave` as *Directory*
5. In *Build Environment*, check *Build inside a Docker container*
6. Select *Pull docker image from repository* and write your image use to Build
  - jenkins-slave-nodejs
  - jenkins-slave-java
7. 	In *Docker installation*, select *docker*
8. On *Volumes*, click *Add* and Add
   ```
   Path on host: /usr/share/jenkins_slave
   Path inside container: /usr/share/jenkins_slave
   ```
 9. Add Build step for your build command
