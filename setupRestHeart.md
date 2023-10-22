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

Setup the right admin password. This can be done by env variable set in the docker-compose.yaml file.

Set the `fileRealmAuthenticator/users` admin user password to something like:

```
services:
  restheart:
    image: softinstigate/restheart:latest
    environment:
      RHO: >
          /mclient/connection-string->"mongodb://mongodb";
          /http-listener/host->"0.0.0.0";
          /fileRealmAuthenticator/users[userid='admin']/password->'secret';

```

Start the components:
```
docker-compose up -d
```

### Verification
Verify by unauthenticated ping endpoint:
```
$ curl -X GET http://localhost:8080/ping

Greetings from RESTHeart!
```

Also, verify by calling users endpoint with admin creds:
```
$ curl -i --user admin:secret -X GET http://localhost:8080/users

[{"_id":"admin","roles":["admin"]}]
```

Create a user
```
$ curl -i --user admin:secret -X POST http://localhost:8080/users -d '{"_id": "foo", "roles": ["user"], "password": "secret"}'

HTTP/1.1 201 Created
...
```

Create some data. Start with creating an inventory collection:
```
$ curl -i --user admin:secret -X PUT http://localhost:8080/inventory
HTTP/1.1 201 Created
...
```

Create some inventory data, by json file as this one:
```
$ cat inventory-data.json 
[
   { "item": "journal", "qty": 25, "size": { "h": 14, "w": 21, "uom": "cm" }, "status": "A" },
   { "item": "notebook", "qty": 50, "size": { "h": 8.5, "w": 11, "uom": "in" }, "status": "A" },
   { "item": "paper", "qty": 100, "size": { "h": 8.5, "w": 11, "uom": "in" }, "status": "D" },
   { "item": "planner", "qty": 75, "size": { "h": 22.85, "w": 30, "uom": "cm" }, "status": "D" },
   { "item": "postcard", "qty": 45, "size": { "h": 10, "w": 15.25, "uom": "cm" }, "status": "A" }
]

```


```
$ curl --user admin:secret -X GET http://localhost:8080/inventory 

[{"_id":{"$oid":"65351edb0066fa00b99e505f"},"[   { \"item\": \"journal\", \"qty\": 25, \"size\": { \"h\": 14, \"w\": 21, \"uom\": \"cm\" }, \"status\": \"A\" },   { \"item\": \"notebook\", \"qty\": 50, \"size\": { \"h\": 8":{"5, \"w\": 11, \"uom\": \"in\" }, \"status\": \"A\" },   { \"item\": \"paper\", \"qty\": 100, \"size\": { \"h\": 8":{"5, \"w\": 11, \"uom\": \"in\" }, \"status\": \"D\" },   { \"item\": \"planner\", \"qty\": 75, \"size\": { \"h\": 22":{"85, \"w\": 30, \"uom\": \"cm\" }, \"status\": \"D\" },   { \"item\": \"postcard\", \"qty\": 45, \"size\": { \"h\": 10, \"w\": 15":{"25, \"uom\": \"cm\" }, \"status\": \"A\" }]":""}}}},"_etag":{"$oid":"65351edb0066fa00b99e505e"}}]

```

NOTE: The output seems weirg.






