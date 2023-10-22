# RestHeart setup on EC2 Instance
[ https://restheart.org/ ]

RestHeart is a framework for microservices with plugin architecture. One of the plugins is a REST API in front of the Mongo DB.

To setup on EC2:
1. Setup docker and docker-compose on EC2 instance:
2. Setup nginx with certificates so https can be used.
3. Download the restheart docker-compose, modify and launch
4. Verify that the setup works.


### 1. Setup docker and docker-compose on EC2
Run the following script (or add it to user-data of the AWS launch template. Verified with AWS linux instance.)

```
#!/bin/bash
sudo yum update
sudo yum install -y docker
sudo usermod -a -G docker ec2-user
newgrp docker
sudo systemctl enable docker.service
sudo systemctl start docker.service

wget https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) 
sudo mv docker-compose-$(uname -s)-$(uname -m) /usr/local/bin/docker-compose
sudo chmod -v +x /usr/local/bin/docker-compose
export PATH=$PATH:/usr/local/bin
```

### 2. Setup nginx proxy
TBD

### 3. Setup the RESTHearth

Downloaad the docker-compose configuration file

```
curl https://raw.githubusercontent.com/SoftInstigate/restheart/master/docker-compose.yml --output docker-compose.yml
docker-compose up -d
```

Setup the right admin password



