# RestHeart setup on EC2 Instance
[ https://restheart.org/ ]

RestHeart is a framework for microservices with plugin architecture. One of the plugins is a REST API in front of the Mongo DB.

To setup on EC2:

1. Start EC2 instance of AWS linux, and enable inbound 443 and 22 connections.
1. Setup docker and docker-compose on EC2 instance:
1. Download the restheart docker-compose, modify and launch
1. Verify that the setup works.


### 2. Setup docker and docker-compose on EC2
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

mkdir etc
```

### 3. Setup the RESTHearth

Downloaad the docker-compose configuration file

```
curl https://raw.githubusercontent.com/SoftInstigate/restheart/master/docker-compose.yml --output docker-compose.yml
docker-compose up -d
```

and modify the `docker-compose.yml` to configure the ssl, mount the right folder for certs and expose the right ports. The restheart service should look like this:

```
services:
  restheart:
    image: softinstigate/restheart:latest
    environment:
      RHO: >
          /mclient/connection-string->"mongodb://mongodb";
          /https-listener/enabled->true;
          /https-listener/host->"0.0.0.0";
          /https-listener/port->4443;
          /https-listener/keystore-path->"./etc/adambures.cz.jks";
          /https-listener/keystore-password->"your-secret";
          /https-listener/certificate-password->"your-secret";

    volumes:
      - /home/ec2-user/etc:/opt/restheart/etc

    depends_on:
      - mongodb
      - mongodb-init

    ports:
      - "8080:8080"
      - "443:4443"
```
see the docker-compose.yml in this folder as an example.

Create the ssl certificates. For that you need JRE installed for the keytool that crates the java keystore. 

Let's abuse for that the docker image of restheart since we will need it anyway.

```
$ docker pull softinstigate/restheart:latest
$ docker run -it --entrypoint -v /home/ec2-user/etc:/opt/restheart/etc /bin/bash softinstigate/restheart
```

This brings us to the shell of the restheart image, with mounted `./etc` folder. Get the provided script and proceed to generate the self signed certificates, storing them into the `etc` folder.


```
# wget https://raw.githubusercontent.com/SoftInstigate/restheart/master/core/bin/generate-certauthority-and-keystore.sh

# chmod u+x generate-certauthority-and-keystore.sh
# ./generate-certauthority-and-keystore.sh --domain adambures.cz -p "yourpassword" -a ./etc

Generate the Certificate Authority private key
Create and self sign the Root Certificate
Certificate Authority certificate (TO BE IMPORTED IN BROWSER): ./etc/devCA.pem
Create a certificate-signing request
Generate the certificate
Certificate request self-signature ok
subject=CN = adambures.cz
Convert certificate to PKCS 12 archive
Import certificates into a keystore file.
Importing keystore ./etc/adambures.cz.p12 to ./etc/adambures.cz.jks...
Done: keystore ./etc/adambures.cz.jks, CA root certificate: ./etc/devCA.pem
```


Start the components:
```
docker-compose up -d
```

### 4. Verification
Verify by unauthenticated ping endpoint:
```
$ curl -k https://localhost:443/ping

Greetings from RESTHeart!
```

First, change the admin's password:
```
$ curl -i -k --user admin:secret -X PATCH https://localhost/users/admin -H "Content-Type: application/json" -d '{ "password": "yourStrongPassword" }'
```

Also, verify by calling users endpoint with admin creds:
```
$ curl -i --user admin:secret -X GET https://localhost/users

[{"_id":"admin","roles":["admin"]}]
```

Create a user
```
curl -i --user admin:secret -H 'Content-Type: application/json' -X POST https://localhost/users -d '{"_id": "foo", "roles": ["user"], "password": "secret"}'

HTTP/1.1 201 Created
...
```

Create some data. Start with creating an inventory collection:
```
$ curl -i --user admin:secret -X PUT https://localhost/inventory
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

$ curl -i --user admin:secret -H 'Content-Type: application/json'  -X POST http://localhost:8080/inventory -d @inventory-data.json
HTTP/1.1 200 OK
...
```


```
$ curl --user admin:secret -X GET http://localhost:8080/inventory 

[{"_id":{"$oid":"65351edb0066fa00b99e505f"},"[   { \"item\": \"journal\", \"qty\": 25, \"size\": { \"h\": 14, \"w\": 21, \"uom\": \"cm\" }, \"status\": \"A\" },   { \"item\": \"notebook\", \"qty\": 50, \"size\": { \"h\": 8":{"5, \"w\": 11, \"uom\": \"in\" }, \"status\": \"A\" },   { \"item\": \"paper\", \"qty\": 100, \"size\": { \"h\": 8":{"5, \"w\": 11, \"uom\": \"in\" }, \"status\": \"D\" },   { \"item\": \"planner\", \"qty\": 75, \"size\": { \"h\": 22":{"85, \"w\": 30, \"uom\": \"cm\" }, \"status\": \"D\" },   { \"item\": \"postcard\", \"qty\": 45, \"size\": { \"h\": 10, \"w\": 15":{"25, \"uom\": \"cm\" }, \"status\": \"A\" }]":""}}}},"_etag":{"$oid":"65351edb0066fa00b99e505e"}}]

```

NOTE: Do not forget to keep sending the content type header when sending json data, by `-H 'Content-Type: application/json'`
NOTE: So far user admin was used for creating and reading data, this has to be changed.


### 5. Setup user's ACL for basic operations
For the user ACL check the examples here:
https://github.com/SoftInstigate/restheart/blob/master/examples/example-conf-files/acl.json

For our purpose let's me create an ACL allowing anyone with role `user` to create, mofify and read documents in his own collection, 
corresponding to his id:

```
cat user-acl.json

[{
    "_id": "userCanGetOwnCollection",
    "description": [
        "**** DESCRIPTION PROPERTY IS NOT REQUIRED, HERE ONLY FOR DOCUMENTATION PURPOSES",
        "allow role 'user' GET document from /{userid}",
        "a read filter apply, so only document with status=public or author=userid are returned <- readFilter",
        "the property 'log' is removed from the response <- projectResponse",
        "NOTE: the id of the user is @user.userid with fileRealmAuthenticator and @user._id with mongoRealmAuthenticator"
    ],
    "roles": ["user"],
    "predicate": "method(GET) and path-template('/{userid}') and equals(@user._id, ${userid})",
    "priority": 100,
    "mongo": {
      "readFilter": {
        "_$or": [{ "status": "public" }, { "author": "@user._id" }]
      },
      "projectResponse": { "log": 0 }
    }
  },
  {
    "_id": "userCanCreateDocumentsInOwnCollection",
    "description": [
        "**** DESCRIPTION PROPERTY IS NOT REQUIRED, HERE ONLY FOR DOCUMENTATION PURPOSES",
        "allow role 'user' to create documents under /{userid}",
        "the request content must contain 'title' and 'content' <- bson-request-contains(title, content)",
        "the request content cannot contain any property other than 'title' and 'content' <- bson-request-whitelist(title, content)",
        "no qparams can be specified <- qparams-whitelist()",
        "the property 'author' and 'status' are added to the request at server-side <- mergeRequest",
        "the property 'log' with some request values is added to the request at server-side <- mergeRequest",
        "NOTE: the id of the user is @user.userid with fileRealmAuthenticator and @user._id with mongoRealmAuthenticator"
    ],
    "roles": ["user"],
    "priority": 100,
    "predicate": "method(POST) and path-template('/{userid}') and equals(@user._id, ${userid}) and bson-request-whitelist(title, content) and bson-request-contains(title, content) and qparams-whitelist()",
    "mongo": {
      "mergeRequest": {
        "author": "@user._id",
        "status": "draft",
        "log": "@request"
      }
    }
  },
  {
    "_id": "userCanModifyDraftsInOwnCollection",
    "description": [
        "**** DESCRIPTION PROPERTY IS NOT REQUIRED, HERE ONLY FOR DOCUMENTATION PURPOSES",
        "allow role 'user' to modify documents under /{userid}",
        "the request content must contain 'title' and 'content' or 'status' <- (bson-request-contains(title, content) or bson-request-contains(status))",
        "the request content cannot contain any property other than 'title', 'content' and 'status' <- bson-request-whitelist(title, content, status)",
        "no qparams can be specified <- qparams-whitelist()",
        "the property 'author' is added to the request at server-side <- mergeRequest",
        "a write filter applies so that user can only modify document with author=userid <- writeFilter",
        "NOTE: the id of the user is @user.userid with fileRealmAuthenticator and @user._id with mongoRealmAuthenticator"
    ],
    "roles": ["user"],
    "priority": 100,
    "predicate": "method(PATCH) and path-template('/{userid}/{docid}') and equals(@user._id, ${userid}) and bson-request-whitelist(title, content, status) and (bson-request-contains(title, content) or bson-request-contains(status)) and qparams-whitelist()",
    "mongo": {
      "mergeRequest": { "author": "@user._id" },
      "writeFilter": { "status": "draft" }
    }
  }
]

$ curl -i --user admin:secret -X POST https://localhost/acl -H 'Content-Type: application/json' -d @user-acl.json

```
