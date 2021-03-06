# Parse Install on CentOS 7

2016-11-25

## Initial configuration

- base system insatlled with: CentOS-7-x86_64-Minimal-1511.iso (sha1: 783eef50e1fb91c78901d0421d8114a29b998478)
- To perform a full system update, run this as root: yum update  (may need to reboot afterwards)
- edit: /etc/hostname /etc/hosts /etc/sysconfig/network
- adduser parseguy
- passwd parseguy
- usermod -aG wheel devops (so that the 'devops' user can use sudo)


## Install via Yum

- to what packages are currently installed: yum list installed *or* yum list installed | grep whatever
- yum install screen
- yum install wget
- yum install net-tools
- yum install vim-enhanced
- yum install git
- yum install policycoreutils-python   (this is for semanage, which is needed for mongodb server configuration)
- yum install telnet (optional, to verify open ports and send raw commands)

## Install via EPEL

- enables you to install packages not in the base CentOS distribution
- https://fedoraproject.org/wiki/EPEL
- yum install epel-release
- yum install smem    (to see system memory usage, you can then run smem -tw)

## Install Nginix

- https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-centos-7
- yum install nginx
- edit: /etc/nginx/nginx.conf
- within the server{} stanza add the following (note the mutiple apps and different their port numbers, 133x):

```
location /parse-testerdb/ {
    proxy_pass http://127.0.0.1:1337;
}

location /parse-app2/ {
    proxy_pass http://127.0.0.1:1338;
}

# for the parse dashboard...
location /dashboard-for-parse/ {
	proxy_pass http://127.0.0.1:4040;
}
```

- systemctl start nginx
- firewall-cmd --permanent --zone=public --add-service=http 
- firewall-cmd --permanent --zone=public --add-service=https
- firewall-cmd --zone=public --add-port=4040/tcp **don't do this in production, this is only to directly access the dashboard; also append --allowInsecureHTTP=1 to the 'parse-dashboard' command below**
- semanage port -a -t http_port_t -p tcp 4040 **don't do this in production**
- firewall-cmd --reload
- firewall-cmd --zone=public --list-all # (to view all open ports)
- iptables -S (to view all firewall rules)
- systemctl enable nginx  (start nginix when system boots)

- To allow nginx to connect to the proxy_pass services: http://stackoverflow.com/a/31403848/452281
- setsebool httpd_can_network_connect on
- setsebool httpd_can_network_connect on -P (persist across reboots)
- getsebool -a | grep httpd | grep on$ # (to verify these changes)


## Install MongoDB

- https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/
- Create a /etc/yum.repos.d/mongodb-org-3.2.repo file so that you can install MongoDB directly, using yum
- Inside this file, create the [mongodb-org-3.2] section
- yum install mongodb-org
- semanage port -a -t mongod_port_t -p tcp 27017
- service mongod start
- systemctl enable mongod (ensure that MongoDB will start following a system reboot)
- database files stored in: /var/lib/mongodb
- log files stored in: /var/log/mongodb

## Install Node.js

- https://github.com/nodesource/distributions#rpm
- wget https://rpm.nodesource.com/setup_6.x && chmod 700 setup_6.x
- verify the contents of this file and then run:
- ./setup_6.x   # (this creates a file: /etc/yum.repos.d/nodesource-el.repo)
- yum install nodejs

## Global Install of Parse Server, Parse Dashboard

- npm install -g parse-server # (this creates many files under /usr/lib/node_modules/ and also creates /usr/bin/parse-server)

## Parse Server Basics

- See also: https://stackoverflow.com/questions/23948527/13-permission-denied-while-connecting-to-upstreamnginx
- parse-server --appId APPLICATION_ID --masterKey MASTER_KEY --databaseURI mongodb://localhost/test
- You can use any arbitrary string as your application id and master key. These will be used by your clients to authenticate with the Parse Server.

## Example Scripts

- Create start_parse_server.sh

```bash
#!/bin/bash

APPNAME=testerdb
APPID="appid123456"
MASTERKEY="masterkey654321"
DBURI="mongodb://127.0.0.1:27017/${APPNAME}?ssl=false" # change to ssl=true once your MongoDB instance is using encryption
PORT=1337
MOUNT=/parse-${APPNAME}

while [ 1 ] ; do
        parse-server --verbose --appId ${APPID} --masterKey ${MASTERKEY} --databaseURI ${DBURI} --port ${PORT} --mountPath ${MOUNT}
        echo "will restart parse-server in 7 seconds..."
        sleep 7
done
```

- For your 2nd app, you will need to create a similar shell script, but change APPNAME, APPID, MASTERKEY, PORT, and MOUNT. DBURI should be OK.
- These names must correspond to the *location* settings in your *nginix.conf* file
- Create write_to_local_parse_server.sh

```bash
#!/bin/bash

APPID="appid123456"
curl -X POST -H "X-Parse-Application-Id: ${APPID}" -H "Content-Type: application/json" -d '{"age":43,"name":"John","location":"Athens"}' http://localhost:1337/parse/classes/testerdb
```

- Create read_from_local_parse.sh

```bash
#!/bin/bash

APPID="appid123456"
curl -X GET -H "X-Parse-Application-Id: ${APPID}" -H "Content-Type: application/json" http://localhost:1337/parse/classes/testerdb
```

- as the parseguy user:

```
[parseguy@staging1 ~]$ ./start_parse_server.sh
appId: appid123456
masterKey: ***REDACTED***
port: 1337
databaseURI: mongodb://127.0.0.1:27017/testerdb?ssl=false
mountPath: /parse
maxUploadSize: 20mb
verbose: true
serverURL: http://localhost:1337/parse

[3262] parse-server running on http://localhost:1337/parse

```

```
[parseguy@staging1 ~]$ ./write_to_local_parse_server.sh
{"objectId":"zMeGUR5NMr","createdAt":"2016-09-14T00:29:45.249Z"}
[parseguy@staging1 ~]$ ./read_from_local_parse_server.sh
{"results":[{"objectId":"zMeGUR5NMr","age":43,"name":"John","location":"Athens","createdAt":"2016-09-14T00:29:45.249Z","updatedAt":"2016-09-14T00:29:45.249Z"}]}
```

- verify you can access the parse server from a remote host,
- Create read_from_remote_parse.sh

```bash
#!/bin/bash

SERVER=192.168.1.27
APPID=appid123456
curl -X GET -H "X-Parse-Application-Id: ${APPID}" -H "Content-Type: application/json" http://${SERVER}/parse/classes/testerdb
```

- The above command should return something like:

```json
{"results":[{"objectId":"zMeGUR5NMr","age":43,"name":"John","location":"Athens","createdAt":"2016-09-14T00:29:45.249Z","updatedAt":"2016-09-14T00:29:45.249Z"}]}
```

- verify you can remotely submit new data to the server, 
- Create readwrite_remote_parse.sh

```bash
#!/bin/bash

SERVER=192.168.1.27
APPID=appid123456

echo; echo "initial read..."
curl -X GET -H "X-Parse-Application-Id: ${APPID}" -H "Content-Type: application/json" http://${SERVER}/parse/classes/testerdb
echo; echo "submit new entry..."
curl -X POST -H "X-Parse-Application-Id: ${APPID}" -H "Content-Type: application/json" -d '{"age":19,"name":"Bobby","location":"Boise"}' http://${SERVER}/parse/classes/testerdb
echo; echo "verify write..."
curl -X GET -H "X-Parse-Application-Id: ${APPID}" -H "Content-Type: application/json" http://${SERVER}/parse/classes/testerdb
```


## Parse Dashboard

- https://github.com/ParsePlatform/parse-dashboard
- npm install -g parse-dashboard
- Create: /home/parseguy/parse-dboard-cfg.json,  Note: the *serverURL* will be used by the browser and therefore should use an IP address that is externally accessible
- change serverURL to https:// if you are using SSL certificates

```json
{
  "apps": [
    {
      "serverURL": "http://192.168.1.27/parse-testerdb",
      "appId": "appid123456",
      "masterKey": "masterkey654321",
      "appName": "TesterDB",
      "iconName": ""
    },
    {
      "serverURL": "http://192.168.1.27/parse-app2",
      "appId": "appidABCDEFG",
      "masterKey": "masterkeyUVWXYZ",
      "appName": "App2"
    }
  ],
  "iconsFolder": "icons",
  "users": [
    {
       "user":"tom",
       "pass":"change me (see below)"
    },
    {
       "user":"jerry",
       "pass":"change me (see below)"
    }
  ],
  "useEncryptedPasswords": true
}
```

- to use encrypted passwords in the parse-dboard-cfg.json file:
- mkdir dashpass && cd dashpass
- Create gen_bcrypt_hash.js (listed below)
- ./gen_bcrypt_hash.js
- copy / paste results into your `parse-dboard-cfg.json` file for the `pass` parameter

```js
#!/bin/node

'use strict';

function install_mod(module_name) {
        const spawn = require( 'child_process' ).spawnSync,

        m = spawn( 'npm', [ 'install', module_name ] );
        console.log( `stderr: ${m.stderr.toString()}` );
        console.log( `stdout: ${m.stdout.toString()}` );
}

function main() {
        install_mod("bcrypt");
        install_mod("prompt");

        var b = require("bcrypt");
        var p = require("prompt");

        var schema = { properties: { password: { message: 'Enter password', hidden: true } } };
        p.start()

        p.get(schema, function(err, result) {
                var hash = b.hashSync(result.password, 15);
                console.log(hash);
                result.password="x";
                hash="x";
        });
}

main();

```



- Create start_dashboard.sh:

```bash
#!/bin/bash

export DASH=/home/parseguy/parse-dboard-cfg.json
export DEBUG="express:*" # (optional)

# (--allowInsecureHTTP=1 is only for initial install and testing)
# NOTE: mountPath needs to be the same as nginx.conf location 
while [ 1 ] ; do
        while [ 1 ] ; do
                echo "`date`: waiting for parse server to start..."
                PARSE=`netstat -a -n | egrep -c ":133[0-0].*LISTEN"`
                if [ "${PARSE}" == "1" ] ; then
                        break
                fi
                sleep 2
        done

        echo "`date`: starting dashboard..."
        sleep 2


        parse-dashboard --config ${DASH} --port 4040 --mountPath /ag-parse-dashboard
        echo "will restart parse-dashboard in 10 seconds..."
        sleep 10
done
```

- As the parseguy user, run ./start_parse_server.sh
- Then run ./start_dashboard.sh


## Run Parse and the Dashboard at system boot

- Create /home/parseguy/screen_boot.rc

```bash
startup_message off
defscrollback 10000

screen -t ParseServer /home/parseguy/start_parse_server.sh
# if you have a second app:
# screen -t ParseServer /home/parseguy/start_parse_server2.sh
screen -t ParseDashboard /home/parseguy/start_dashboard.sh
screen -t ParseBash /bin/bash
```

- run crontab -e and add the following line:
- @reboot /bin/screen -d -m -c /home/parseguy/screen_boot.rc


## Install SSL Certificate for Nginix

- yum install certbot
- certbot certonly
- follow https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04
- (making adjustments for CentOS instead of Ubuntu)
- For the root crontab file, use: 
- 45 2 * * * /bin/certbot renew --quiet
- 50 2 * * * /sbin/service nginx restart



## Install SSL Certificate for MongoDB

- Adapted from this guide, "MongoDB 3.2.x SSL with Letsencrypt": https://gist.github.com/leommoore/1e773a7d230ca4bbe1c2
- Do not install the LetsEncrypt software with this method
- Instead, start at the "download IdenTrust DST" section
- install_mongo_ssl.sh
```bash
#!/bin/bash

# adapted from: https://gist.github.com/leommoore/1e773a7d230ca4bbe1c2

SOURCE=/etc/letsencrypt/live/parse.example.com
DEST=/etc/ssl/mongodb

if [ ! -e ${DEST} ] ; then
    mkdir -m 700 ${DEST}
    chown mongod:mongod ${DEST}
else
    rm -f ${DEST}/ca.* ${DEST}/mongodb.pem
fi

cat ${SOURCE}/privkey.pem ${SOURCE}/fullchain.pem > ${DEST}/mongodb.pem

if [ ! -e ${DEST}/ca.crt ] ; then
    # from: https://www.identrust.com/certificates/trustid/root-download-x3.html
    echo "-----BEGIN CERTIFICATE-----" > ${DEST}/ca.crt
    echo "MIIDSjCCAjKgAwIBAgIQRK+wgNajJ7qJMDmGLvhAazANBgkqhkiG9w0BAQUFADA/" >> ${DEST}/ca.crt
    echo "MSQwIgYDVQQKExtEaWdpdGFsIFNpZ25hdHVyZSBUcnVzdCBDby4xFzAVBgNVBAMT" >> ${DEST}/ca.crt
    echo "DkRTVCBSb290IENBIFgzMB4XDTAwMDkzMDIxMTIxOVoXDTIxMDkzMDE0MDExNVow" >> ${DEST}/ca.crt
    echo "PzEkMCIGA1UEChMbRGlnaXRhbCBTaWduYXR1cmUgVHJ1c3QgQ28uMRcwFQYDVQQD" >> ${DEST}/ca.crt
    echo "Ew5EU1QgUm9vdCBDQSBYMzCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB" >> ${DEST}/ca.crt
    echo "AN+v6ZdQCINXtMxiZfaQguzH0yxrMMpb7NnDfcdAwRgUi+DoM3ZJKuM/IUmTrE4O" >> ${DEST}/ca.crt
    echo "rz5Iy2Xu/NMhD2XSKtkyj4zl93ewEnu1lcCJo6m67XMuegwGMoOifooUMM0RoOEq" >> ${DEST}/ca.crt
    echo "OLl5CjH9UL2AZd+3UWODyOKIYepLYYHsUmu5ouJLGiifSKOeDNoJjj4XLh7dIN9b" >> ${DEST}/ca.crt
    echo "xiqKqy69cK3FCxolkHRyxXtqqzTWMIn/5WgTe1QLyNau7Fqckh49ZLOMxt+/yUFw" >> ${DEST}/ca.crt
    echo "7BZy1SbsOFU5Q9D8/RhcQPGX69Wam40dutolucbY38EVAjqr2m7xPi71XAicPNaD" >> ${DEST}/ca.crt
    echo "aeQQmxkqtilX4+U9m5/wAl0CAwEAAaNCMEAwDwYDVR0TAQH/BAUwAwEB/zAOBgNV" >> ${DEST}/ca.crt
    echo "HQ8BAf8EBAMCAQYwHQYDVR0OBBYEFMSnsaR7LHH62+FLkHX/xBVghYkQMA0GCSqG" >> ${DEST}/ca.crt
    echo "SIb3DQEBBQUAA4IBAQCjGiybFwBcqR7uKGY3Or+Dxz9LwwmglSBd49lZRNI+DT69" >> ${DEST}/ca.crt
    echo "ikugdB/OEIKcdBodfpga3csTS7MgROSR6cz8faXbauX+5v3gTt23ADq1cEmv8uXr" >> ${DEST}/ca.crt
    echo "AvHRAosZy5Q6XkjEGB5YGV8eAlrwDPGxrancWYaLbumR9YbK+rlmM6pZW87ipxZz" >> ${DEST}/ca.crt
    echo "R8srzJmwN0jP41ZL9c8PDHIyh8bwRLtTcm1D9SZImlJnt1ir/md2cXjbDaJWFBM5" >> ${DEST}/ca.crt
    echo "JDGFoqgCWjBH4d1QB7wCCZAA62RjYJsWvIjJEubSfZGL+T0yjWW06XyxV3bqxbYo" >> ${DEST}/ca.crt
    echo "Ob8VZRzI9neWagqNdwvYkQsEjgfbKbYK7p2CNTUQ" >> ${DEST}/ca.crt
    echo "-----END CERTIFICATE-----" >> ${DEST}/ca.crt

    cat ${SOURCE}/chain.pem >> ${DEST}/ca.crt
fi

openssl x509 -in ${DEST}/ca.crt -out ${DEST}/ca.pem -outform PEM
openssl verify -verbose -CAfile ${DEST}/ca.crt ${DEST}/mongodb.pem

chown mongod:mongod ${DEST}/ca.* ${DEST}/mongodb.pem
chmod 600 ${DEST}/ca.* ${DEST}/mongodb.pem

echo
ls -la ${DEST}
```

- service mongod restart
- On a remote server, copy ca.pem and mongodb.pem to ${HOME}/mongodb
- From that system, connect with:
- mongo --ssl -sslCAFile ${HOME}/mongodb/ca.pem --sslPEMKeyFile ${HOME}/mongodb/mongodb.pem parse.example.com:27017/testerdb
- or without the certs:
- mongo -ssl "mongodb://parse.example.com:27017/testerdb"

## Install MongoDB username/password authentication 

- https://docs.mongodb.com/v3.2/tutorial/enable-authentication/
- Connect to your local mongoDB instance:
- mongo --ssl -sslCAFile /etc/ssl/mongodb/ca.pem --sslPEMKeyFile /etc/ssl/mongodb/mongodb.pem --host ${HOSTNAME} --port 27017

```
    use admin
    db.createUser( { user: "mngAdmin", pwd: "StrongAdmPass13579", roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] } )
```

- Edit /etc/mongod.conf and add:

```
    security:
    authorization: enabled
```

- restart mongod: service mongod restart
- verify you can connect:
- mongo --ssl -sslCAFile /etc/ssl/mongodb/ca.pem --sslPEMKeyFile /etc/ssl/mongodb/mongodb.pem --authenticationDatabase admin -u mngAdmin -p StrongAdmPass13579 ${HOSTNAME}:27017/testerdb

- create two users, a read-only user, denoted by RO and a read-write user, denoted by RW
- Do not use these characters in passwords: double-quotes, at sign, colon

```
    use admin
    db.createUser( { user: "mngROtesterdb", pwd: "ReadOnlyPass97531", roles: [{ role:"read", db: "testerdb" } ] } )
    db.createUser( { user: "mngRWtesterdb", pwd: "ReadWritePass0246", roles: [{ role:"readWrite", db: "testerdb" } ] } )
````

- If you need to change a password (see also: https://docs.mongodb.com/v3.2/tutorial/manage-users-and-roles/), run the following command:

```
    use admin
    db.changeUserPassword("mngUser1", "myNewPass57132")
```

- verify you can connect remotely:

```
    mongo -ssl  "mongodb://mngROtesterdb:ReadOnlyPass97531@parse.example.com:27017/testerdb?ssl=true&authSource=admin"
    MongoDB shell version: 3.2.9
    > db.testerdb.find()
    { "_id" : "COJgythRRU", "age" : 43, "name" : "John", "location" : "Athens", "_created_at" : ISODate("2016-10-08T17:35:42.929Z"), "_updated_at" : ISODate("2016-10-08T17:35:42.929Z") }
    { "_id" : "oAqUyxAVU3", "age" : 19, "name" : "Bobby", "location" : "Boise", "_created_at" : ISODate("2016-10-08T17:40:33.196Z"), "_updated_at" : ISODate("2016-10-08T17:40:33.196Z") }
```

- A URL with read-write access will be needed to migrate your app from parse.com
- See also: https://docs.mongodb.com/manual/reference/connection-string/

## Export your data from parse.com

- https://www.parse.com/login
- Use the "new" parse dashboard
- click on your app (on the right side)
- click on app settings -> general (on the left side)
- export data (an email will be sent to you with a download link)
- or you can also use "Clone this app", but this will not clone any data, just the schema
- Example (this is case-sensative in the parse.com Web UI) mongodb://parse.example.com:27017/testerdb?ssl=true


## iOS Examples

```objc
/*
Podfile:

target 'parse-objc' do
    pod 'Parse'
end
*/

// ViewController.m

#import "ViewController.h"
#import "Parse.h"

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    [Parse initializeWithConfiguration:[ParseClientConfiguration configurationWithBlock:^(id<ParseMutableClientConfiguration> configuration) {
        configuration.applicationId = @"appid123456";
        configuration.clientKey = @"";
        configuration.server = @"http://192.168.1.27/parse/";
    }]];
    
    PFQuery *query = [PFQuery queryWithClassName:@"testerdb"];
    // to return a sungle entity...
    //[query whereKey:@"name" equalTo:@"John"];
    [query findObjectsInBackgroundWithBlock:^(NSArray *objects, NSError *error) {
        if (!error) {
            // The find succeeded.
            NSLog(@"Successfully retrieved %ld contacts.", objects.count);
            NSLog(@"========================================================");
            // Do something with the found objects
            for (PFObject *object in objects) {
                NSLog(@"object: %@", object.objectId);
                NSLog(@"web   : %@", object[@"contactWeb"]);
                NSLog(@"phone : %@", object[@"contactPhone"]);
                NSLog(@"---------------------------");
            }
        } else {
            // Log details of the failure
            NSLog(@"Error: %@ %@", error, [error userInfo]);
        }
    }];
    
}

```



## Todo

- import parse.com data and used a username/password for the MongoDB connect string

