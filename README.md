# Rocket.Chat over Debian 8

Little toolset:  

```
su root
apt-get update 
apt-get install wget curl htop sudo screen
```

# Install Mongo 3.2

```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927

echo "deb http://repo.mongodb.org/apt/debian wheezy/mongodb-org/3.2 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list

sudo apt-get update

sudo apt-get install -y mongodb-org curl graphicsmagick

```

## Pin Versions

```
echo "mongodb-org hold" | sudo dpkg --set-selections
echo "mongodb-org-server hold" | sudo dpkg --set-selections
echo "mongodb-org-shell hold" | sudo dpkg --set-selections
echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
echo "mongodb-org-tools hold" | sudo dpkg --set-selections
```

## Conf Mongo

Configure /etc/hosts and /etc/hostname:

```
192.168.25.107 	m1.dorgam.it m1
192.168.25.108 	m2.dorgam.it m2
192.168.25.109 	m3.dorgam.it m3
```

hostname:
  
```
m1.dorgam.it
```

Repetir essas configurações nas 3 instancias.

`reboot`  

 /etc/mongod.conf  

```
sudo nano /etc/mongod.conf
```
Edit the ReplicaSet Name and save:  

```
replication:
      replSetName:  "001-rs"
```

`sudo service mongod restart`  

```
mongo
> rs.initiate()

{
        "info1" : "no configuration specified. Using a default configuration f
or the set",
        "me" : "rocket:27017",
        "ok" : 1
}
 
001-rs:PRIMARY> rs.add("m2.dorgam.it:27017");
001-rs:PRIMARY> rs.add("m3.dorgam.it:27017");
``` 

```
echo 'export MONGO_OPLOG_URL=mongodb://localhost:27017/local' >> ~/.bashrc  
source ~/.bashrc  
```


## Install NodeJs

```
apt-get install build-essential wget sudo htop screen nano curl

curl -sL https://deb.nodesource.com/setup_4.x | sudo -E bash -
sudo apt-get install -y nodejs

```
## Install Rocket.Chat servers

```
sudo su
cd /opt
wget https://cdn-download.rocket.chat/build/rocket.chat-develop.tgz
tar zxvf rocket.chat-develop.tgz
rm rocket.chat-develop.tgz
mv bundle/ rocket/
cd rocket/programs/server
npm install
cd ../../

nano /etc/hosts
> 192.168.25.107  m1.dorgam.it m1

sudo ROOT_URL=http://chat.dorgam.it/ \
    MONGO_URL=mongodb://m1.dorgam.it:27017/rocketchat \
    PORT=80 \
    node main.js
```

**Rocket.Chat Is Running!** faça o mesmo nos outros nódulos.

## Install Nginx

`apt-get update && apt-get install nginx`  

`nano /etc/nginx/site-enable/default`

```
server {
	listen 80;
	server_name chat.dorgam.it;
	rewrite ^/(.*) https://chat.dorgam.it/$1 permanent;
}

server {
	listen 443;
	server_name chat.dorgam.it;
	error_log /var/log/nginx/rocketchat.access.log;

	ssl on;
	ssl_certificate /etc/nginx/certificados/certificado.crt;
	ssl_certificate_key /etc/nginx/certificados/chave_privada.key;

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # don’t use SSLv3 ref: POODLE

	location / {
		proxy_pass http://chat.dorgam.it:3000/;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
		proxy_set_header Host $http_host;

		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forward-Proto http;
		proxy_set_header X-Nginx-Proxy true;

		proxy_redirect off;
	}
}
```

`service nginx restart`  
