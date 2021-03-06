### List images
```
docker images
Tag Image: docker image tag <source_Image_name> <dest_Image_name:tag>

Push Image: docker image push <image_Name>

Cleanup unused Volumes: docker image prune
```
### List images with size 
```
docker images | awk '{print $1"\t"$2"\t"$7" "$8}'
```
### list running containers 
```
docker ps
```
### list all running containers
```
docker ps -a
```
### stop all containers
```
docker stop $(docker ps -a -q)
```
### delete all containers
```
docker rm $(docker ps -a -q)
```
### delete image
```
docker rmi <image name>
```
### delete all images
```
docker rmi $(docker images -a -q)
```
### attach to container
```
docker exec -t -i <container id> /bin/bash
```
### To get an ip 
```
docker inspect <id> | grep IPAddr

Containers

List all Containers: docker container ls

Start a new Container: docker container run <image_name>

docker run <image_name>

Start a new container interactively: docker container run -it <image_name> <command>

Run additional command in existing container: docker container exec -it <container_name> <command>

Stop a single container: docker container stop <container_id>

Stop multiple containers: docker container stop <container_id1> <container_id2> <container_id3> ...

Stop / Remove all Containers: docker stop $(docker ps -a -q)

Container Logs: docker container logs <container_id>

Container Config: docker container inspect <container_id>

Performance stats for one container: docker container stats <container_name>

Performance stats for all containers: docker container stats

Delete all containers: docker rm $(docker ps -a -q)

 
---
### Networks

List all Docker Networks: docker network ls

Inspect Network: docker network inspect <network_id>

### Volumes

List all Volumes: docker volume ls

Remove volume: docker volume rm

Cleanup unused Volumes: docker volume prune

Additional Commands:

Force Removal: --force, -f
```
### To login into docker container 
```
docker run -it latest:start /bin/bash
apt-get install -y openssh-server
mkdir /var/run/sshd;chmod 0755 /var/run/sshd;/usr/sbin/sshd
useradd --create-home --shell /bin/bash --groups sudo testid
passwd testid
ifconfig | awk '/inet addr/{print substr($2,6)}'
ctrl PQ
docker inspect <containerid> | grep IPAddr
ssh testid@<ip>
```
### To assign port
```
docker pull nginx
docker run -d --name=webserver1 -P nginx:latest
docker ps   see the ports 
elinks https://localhost:32771 or elinks https://<ip>:32771 
docker container inspect webserver1 | grep IPAddr
```
### Work with dockerfile
```
vi Dockerfile
FROM ubuntu
MAINTAINER mgang11 
RUN apt-get update && RUN apt-get upgrade -y && apt-get install -y apache2 telnet elinks openssh-serevr
CMD ["/usr/sbin/sshd","-D","FOREGROUND"]
docker build -t latest mgang11/myapache .
docker images
docker run -it mgang11/myapache:latest /bin/bash
```

# Dockerswarm
### Setting up Swarm 
```
get ip of node <ifconfig>
docker swarm init --advertise-addr <private-ip>  : it'll also give worker token to add into cluster
docker swarm join-token manager                  : it'll also give manager token to add into cluster

```
### To see the worker token from Master and add node into cluster
```
docker swarm join-token worker
docker node ls : Run from master to see leader or nodes
docker system info : to see swarm status, Manager and node counts

docker swarm join --token <tokenid from master> :  run it on each node
```

# Docker service Create
### Add 1 Manager and 2 nodes into cluster
```
docker service create --name webapp -p 80:80 httpd
docker service ls 
docker service ps webapp
elinks http://node1 or node2 or node3 will work cause it useses the mesh routing 

```
# Docker service scale up 
### add replicas
```
docker service ps webapp
docker service update --replicas 3 webapp
docker service ps webapp : should see 3 replicas running on cluster
docker service update --replicas 10 --detach=false webapp
docker service ps webapp : it'll spin up into 10 replica
docker service update --replicas 2 webapp 
docker service ps webapp 
docker service update --limit-cpu=.5 --reserve-cpu=1 --limit-memory=128m --reserve-memory=256m webapp
docker service ps webapp   : you'll see one going down 

docker service create --name nginxapp -p 7101:80 nginxapp
elinks http://<ip>
docker service ls : should see two apps webapp and nginxapp

docker service scale --detach=false webapp=4 nginxapp=2

```

# Docker-Compose
### build
```
assuming nginx and https images are there 
mkdir Dockerfile;cd Dockerfile
vi Docker-compose.yml
version: '3'
services: 
  webapi1:
    image: httpd:v1
    build: .
    ports:
      - "3281:80"
 webapi2:
   images: httpd:v1
    ports:
      - "3282:80"
 load-balancer:
   image: nginx:v1
   ports:
     - "3280:80"    
     
docker-compose up -d

docker ps   : you'll see three process running
docker-compose ps
elinks https://<ip>:3281 or without ip
docker-compose ps 
docker-compose down --volumes
deploy swarm from the same compose file
docker stack deploy --compose-file docker-compose.yml stackapp
docker service ls
```

### build (no cache)
```
docker-compose build --force-rm --no-cache
```

### run containers
```
docker-compose up
```

# Docker-Machine
### Create
```
docker-machine create --driver [virtualbox, amazonec2, digitalocean] <machine name>
```
### Regenerate user certificates
```
docker-machine regenerate-certs <machine name>
```
### list mahcines
```
docker-machine ls
```
### switch to machine terminal
```
eval "$(docker-machine env <machine name>)"
```
### connect to existing host
```
docker-machine create --driver generic --generic-ip-address <ip> --generic-ssh-user <username> --generic-ssh-key <key-location> <machine name>
```

https://github.com/SummitRoute/aws_managed_policies

aws iam list-policies > list-policies.json
cat list-policies.json | jq -cr '.Policies[] | select(.Arn | contains("iam::aws"))|.Arn +" "+ .DefaultVersionId+" "+.PolicyName' | xargs -n3 sh -c 'aws iam get-policy-version --policy-arn $1 --version-id $2 > "policies/$3"' sh

https://github.com/powerof-s/AWS-List-EC2-RDS/blob/master/ListECandRDS%20copy.sh

aws sso login --profile <profilename>
echo "######## LISTING EC2 INSTANCES################"

for region in `aws ec2 describe-regions --all-regions --query "Regions[].{Name:RegionName}" --output text --profile <profilename>`
do
     echo -e "\nListing Instances in region:'$region'..."
#aws ec2 describe-instances --region $region
aws ec2 describe-instances --region $region  --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, State.Name, InstanceId]' --output text --profile <profilename> | grep -E '(running|pending)'
done

echo "######## END OF LISTING EC2 INSTANCES##########"

echo "####### LISTING RDS INSTANCES##################"

for region in `aws ec2 describe-regions --all-regions --query "Regions[].{Name:RegionName}" --output text --profile <profilename>`
do
     echo -e "\nListing Instances in region:'$region'..."
aws rds describe-db-instances --region $region --profile <profilename>
#aws ec2 describe-instances --region $region
done

echo "####### END OF LISTING RDS INSTANCES###########"


