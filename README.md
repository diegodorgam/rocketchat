# Rocket.Chat HA Install

This is a detailed tutorial about how to install Rocket.Chat in a High Availability (HA) architeture, over Debian 8.  
Some tips and good pratices that you'll found here can be used in other architetures too, but remember that if you are not going to need a HA architeture, you might wanna use the Docker image to save some precious time.  

So, here is what we're gonna do:

**1. Let's set up a MongoDB replicaSet with 3 servers;**  

> **INFO**: If you are planning to have a realy big number of users, you might want to take a look at MongoDB Sharding tutorial too.

**2. We will install a good and neet Rocket.Chat server;**

> **TIP**: I will be installing a Virtual Machine server, but if you want to be in state of art, you will check the Docker's Swarm architeture, just to get inspired.

**3. A light weight Load Balancer and Reverse Proxy with NGINX;**

> **TOUGHT**: You could do this with apache or other apps just for fun, if you do, please let me know what you find out.

**4. Your Own Jitsi-Meet internal Server;**

> **YEP**: You probably won't need one, you can use the meet.jit.si server for free and it is a awesome server, but if want to have your own internal videoconference tool running inside your NAT, thats the way to go. 

After this tutorial you will need to explore the Rocket Power, learn some good confs, set up live chat in your website, maybe do some webhooks integrations with gitlab and github, and probably you will want to checkou the new Hubot integration. Taking all that in consideration, I won't spoiled out all your fun in this tutorial, but I promise to answer the comments =).

## Prepare your self

First we'll need some tools to start, get inside your servers and start to set a health enviroment. By the way, we will need 7 servers to this tutorial.

```
apt-get update  
apt-get install curl htop sudo screen graphicsmagick
```

Always good to remember `screen`, before anything.

Configure /etc/hosts in your machines:

```
192.168.25.101 	m1.dorgam.it m1 # MongoDB Prymary
192.168.25.102 	m2.dorgam.it m2 # MongoDB replica
192.168.25.103 	m3.dorgam.it m3 # MongoDB replica
192.168.25.121 	r1.dorgam.it r1 # Rocket.Chat Server 1
192.168.25.122 	r2.dorgam.it r2 # Rocket.Chat Server 2
192.168.25.131 	j1.dorgam.it j1 # Jitsi-Meet Single Server
```

Will be nice if you set their /etc/hostname too. MongoDB replicaSet server will need to have that hostname just like this:

**MongoDB Primary**

```
m1.dorgam.it
```
**MongoDB ReplicaSet**

```
m2.dorgam.it
```
**MongoDB ReplicaSet**

```
m3.dorgam.it
```

> **ATTENTION**: I'm using my domain (dorgam.it) to configure cannonical names to the servers, but you obviously can give them the name tha you want, just remember: _they have to be cannonical_. MongoDB ReplicaSet must be able to resolv the names of the servers.


## Install MongoDB 3.2

Let's install MongoDB in version 3.2, directly from the repository. We can use the ubuntu keyserver:  

```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927
```

Then you can...  

```
echo "deb http://repo.mongodb.org/apt/debian wheezy/mongodb-org/3.2 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
```

And of course...

```
sudo apt-get update
sudo apt-get install -y mongodb-org
```

See, not so hard. Now, let's get serious:

```
echo "mongodb-org hold" | sudo dpkg --set-selections
echo "mongodb-org-server hold" | sudo dpkg --set-selections
echo "mongodb-org-shell hold" | sudo dpkg --set-selections
echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
echo "mongodb-org-tools hold" | sudo dpkg --set-selections
```

This will pin your MongoDB packages in the current version, so you can't upgrade them by accident. Rocket.Chat stack is all about sinergy, you don't want to have something running on it with the wrong version.

### Configure MongoDB ReplicaSet

Now, to configure the replicaSet, you must add the property replSetName in all servers with the same name, so the servers will know that they belong to the same group.  
Like this:

```
sudo nano /etc/mongod.conf
```
In the `net:` section, you will need to set an array of IP addresses that will be authorized to bind with the server. In that array we will include all the MongoDB instances and the Rocket.Chat server instances:

```
net:
  port: 27017
  bindIp: [192.168.25.101, 192.168.25.102, 192.168.25.103, 192.168.25.121, 192.168.25.122]
```

In the `replication:` section, insert your replSetName:  

```
replication:
      replSetName:  "001-rs"
```

> **TIP**: In the future, you might wanna add more replica sets in your architeture, that is what the Shardding feature is about, so, in order to be ready to that scenario, should be a good idea to use a number and some letters, like `"001-rs"`.  
>> **ATTENTION**: Remember that `"001-rs"` is **the name of the Replica Set**, not the name of the server inside the set. All MongoDB servers that will be part of this set, must have the same `replSetName` in their `/etc/mongod.conf` files.

Now you just have to restart your mongod service and you're ready to go:

```
sudo service mongod restart
```

Repeat this configurations in all MongoDB instances, they must be able to read the same replSetName and to communicate with each other.

--

NOW, **ONLY in the PRIMARY instance** (m1), you will do the following:

```javascript
# mongo
...
> rs.initiate()

{
        "info1" : "no configuration specified. Using a default configuration f
or the set",
        "me" : "rocket:27017",
        "ok" : 1
}
```

This will initiate the replica set for good, because even having the `replication` section configured in mongod.conf, your MongoDB server wont behave as a replica set until you initiate it.  

If everything goes fine, you will get the `"ok" : 1` message, just like above, otherwise, copy the error and do some research to find out what went wrong.

`OK:1`! Lets put the other instances in the set:
 
```javascript
001-rs:PRIMARY> rs.add("m2.dorgam.it:27017");
001-rs:PRIMARY> rs.add("m3.dorgam.it:27017");
```

You can check `rs.status()` to see if everything went right.

**Congratulations**! Your ReplicaSet is now ready to take some serious load, and keep you safe. Replica Sets give you the hance to not worry with:
- Redundancy and Data Availability
- Replication in MongoDB
- Asynchronous Replication
- Automatic Failover (of Course!)
  
Check the [MongoDB tutorial](https://docs.mongodb.com/manual/replication/) if you want to know more about ReplicaSets.


## Install Rocket.Chat servers

To install Rocket.Chat version 0.42.0 (actual latest) you will need to have a 4.5 Node.JS installation in your server, not 6.x, to ensure that everything runs fine.  
First, in your Rocket.Chat servers (r1 an r2) do the following:

```
sudo su
apt-get install build-essential
curl -sL https://deb.nodesource.com/setup_4.x | sudo -E bash -
apt-get install -y nodejs

```

Now lets set our enviroment variables:

```shell
nano ~/.bashrc 

...

export ROOT_URL=http://chat.dorgam.it/
export MONGO_URL=mongodb://m1.dorgam.it:27017/rocketchat
export MONGO_OPLOG_URL=mongodb://localhost:27017/local
export PORT=80
```

```
source ~/.bashrc
```

And finally lets get our rocket latest package to run inside `/opt` directory:

```
cd /opt
curl -L https://rocket.chat/releases/latest/download -o rocket.chat.tgz
tar zxvf rocket.chat.tgz
rm rocket.chat.tgz
mv bundle/ rocket/
cd rocket/programs/server
npm install
cd ../../
```

Now, we could go by `node main.js` and celebrate our brand new install, but that would be silly because once the server restart, our service wouldn't. In order to have a fail over recovery of our service, we must put it in invoke.rc.d, so we get ourselves a neet init script, wright? YES, but there is a cleaner and better way to do this!  

Just follow these steps:

```
npm install -g pm2
```

Thats it, checkout this [advanced production process manager for Node.js](http://pm2.keymetrics.io/) before you get too excited.

Lets create our `pm2.json` conf file and run it:

```
cd /opt/rocket/

nano pm2.json
```

```json
{
        "apps": [{
                "name": "chat.dorgam.it",
                "log_date_format": "YYYY-MM-DD HH:mm:ss SSS",
                "script": "/opt/rocket/main.js",
                "out_file": "/var/log/rocket/app.log",
                "error_file": "/var/log/rocket/err.log",
                "port": "80",
                "env": {
                        "MONGO_URL": "mongodb://m1.dorgam.it:27017/rocketchat",
                        "MONGO_OPLOG_URL": "mongodb://m1.dorgam.it:27017/local",
                        "ROOT_URL": "http://chat.dorgam.it",
                        "PORT": "80"
                }
        }]
}
```

For the next series, lets start our server (yeeeii), set the `restart` flag on, execute the debian startup script into update-rc.d defaults and save our configurations:

```
pm2 start pm2.json
pm2 restart 'all'
pm2 startup debian

su -c "chmod +x /etc/init.d/pm2-init.sh && update-rc.d pm2-init.sh defaults"

pm2 save

```

**DONE! Rocket.Chat Is Running!** But just on the first instance, we want to have two of them, thats right, two rockets must be launched! So do the same with the other server, and lets get our load balancer to work.

## Install Nginx Load Balancer and Reverse Proxy

What we are going to do now is set a Nginx load balancer in front of our Rocket.Chat servers, to distribute users connections trought our them and, to catch a break, configure some reverse proxy features to ensure that some of the rendering and files transactions will be cached, taking some load off our applications servers and making all this work a little more easy on them.

You can check more about [Nginx load balancing](http://nginx.org/en/docs/http/load_balancing.html) and [reverse proxy](https://www.nginx.com/resources/admin-guide/reverse-proxy/) in their online manuals.

Here, I'm setting a 7th machine `192.168.25.120  chat.dorgam.it` that will respond for the url of my service. 

Now, do the honors:

```
sudo su
apt-get update
apt-get install nginx
```  

Edit the configuration file:

```
nano /etc/nginx/site-enable/default
```

I'll do the comments on the code:

```nginx
upstream chat.dorgam.it{
  ip_hash; #makes it sticky, users request allways returns to the same server 
  server r1.dorgam.it; # remember setting ip address for these guys in the /etc/hosts
  server r2.dorgam.it;
}
server { 
	listen 80; #lets take everything that comes in http 80 and throw to https 443
	server_name chat.dorgam.it;
	rewrite ^/(.*) https://chat.dorgam.it/$1 permanent;
}

server { 
	listen 443; # lets protect our data transactions
	server_name chat.dorgam.it;
	error_log /var/log/nginx/rocketchat.access.log;

	ssl on; # check letsencrypt's certbot for these =)
	ssl_certificate /etc/nginx/certificados/certificado.crt; # you need to create these, use self-signed our a free CA 
	ssl_certificate_key /etc/nginx/certificados/chave_privada.key;
	
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 
	
	location / { # our reverse proxy
		proxy_pass http://chat.dorgam.it/;
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

Save this and restart Nginx:

```
service nginx restart
```  

Now you can access your Rocket.Chat server in HA architecture for the first time, take it to a spin, call your friends and try to take it down. Tell us how many load was necessary, if you succeed.

## Install Jitsi-Meet

So this is a little more simple, the Jitsi-Meet conference server is quite easy to install in debian, all you have to do is [follow these steps](https://github.com/jitsi/jitsi-meet/wiki/Debian-installation), and give a little flavor after.

First add the repository source:

```
echo "deb http://download.jitsi.org/deb unstable/" | sudo tee /etc/apt/sources.list.d/jitsi-meet.list
```
Download the key:

```
wget -qO - https://download.jitsi.org/nightly/deb/unstable/archive.key | apt-key add -
```
And go nuts:

```
sudo su
apt-get update
apt-get install jitsi-meet
```

The installation will ask you about the domain, I used `meet.dorgam.it`, and will ask if you want to use a certificate or create a self-signed one. 
> **PLEASE** [create the certificates using certbot](https://certbot.eff.org/#debianjessie-nginx), do not use self-signed certificates in production enviroment.  
> Give a special attention to the automatic renewal feature.
 
Now, adjust the jitsi-meet screen changing the interface_config.js file:

```
nano /usr/share/jitsi-meet/interface_config.js
```

```javascript
var interfaceConfig = {
    CANVAS_EXTRA: 104,
    CANVAS_RADIUS: 0,
    SHADOW_COLOR: '#ffffff',
    INITIAL_TOOLBAR_TIMEOUT: 20000,
    TOOLBAR_TIMEOUT: 4000,
    DEFAULT_REMOTE_DISPLAY_NAME: "Colega",
    DEFAULT_LOCAL_DISPLAY_NAME: "Eu",
    SHOW_JITSI_WATERMARK: false,
    JITSI_WATERMARK_LINK: "",
    SHOW_BRAND_WATERMARK: false,
    BRAND_WATERMARK_LINK: "",
    SHOW_POWERED_BY: false,
    GENERATE_ROOMNAMES_ON_WELCOME_PAGE: true,
    APP_NAME: "Chat.Caixa",
    INVITATION_POWERED_BY: true,
    // the toolbar buttons line is intentionally left in one line, to be able
    // to easily override values or remove them using regex
    TOOLBAR_BUTTONS: ['authentication', 'microphone', 'camera', 'desktop', 'recording', 'security', 'invite', 'chat', 'etherpa$
    // Determines how the video would fit the screen. 'both' would fit the whole
    // screen, 'height' would fit the original video height to the height of the
    // screen, 'width' would fit the original video width to the width of the
    // screen respecting ratio.
    VIDEO_LAYOUT_FIT: 'both',
    /**
     * Whether to only show the filmstrip (and hide the toolbar).
     */
    filmStripOnly: false,
    RANDOM_AVATAR_URL_PREFIX: false,
    RANDOM_AVATAR_URL_SUFFIX: false,
    FILM_STRIP_MAX_HEIGHT: 120
};

```
> **ATTENTION** JITSI videobridge uses the 10000 UDP port, it must be free so users can have a good experience in video conferences.   

All done, now remember to set your Video Conference parameters in the Rocket.Chat Administration Area.


## Enjoy =)

Hope you've enjoyed, if you have any thoughts on this, please share with us, and have a nice Rocket Launch!