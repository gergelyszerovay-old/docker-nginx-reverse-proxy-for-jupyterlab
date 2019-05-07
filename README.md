# Nginx reverse proxy for JupyterLab Docker containers

If you manage multiple JupyterLab containers for different users on a single Docker server, this reverse proxy container:
* provides https access to multiple JupyterLab containers through a single ip address
* enables username/password authentication for each container

Required tools:
* “apache2-utils” package, it contains the htpasswd command to generate username/password pairs
* “openssl” package to generate self-signed certificates

The sample config files in this repository manages two users (user1 and user2), both of them has a JupyterLab container. 

## Directory structure:
* nginx-reverse-proxy-for-jupyterlab - base directory
  * default.conf - sample Nginx configuration file
  * ssl-certs - folder of ssl certificates (you have to provide/regenerate your own certificates, see below)
    * cert-gen.sh - generates a new self-signed certificate (key.pem and cert.pem) files
    * key.pem - generated certificate file
    * cert.pem - generated certificate file
  * htpasswd - folder for per container authentication
    * user1 - username and password for user1
    * user2 - username and password for user2

## Networking, ips and domains

The Docker server’s external ip is 192.168.99.111. The following domains are assigned to this ip:
* alpine-docker-host
* user1.alpine-docker-host
* user2.alpine-docker-host

The JupyterLab containers are connected to an user-defined Docker network, it’s: name is ‘user-defined’, it’s subnet is 172.18.0.0/16.

user1’s JupyterLab container is bind to the the 172.18.0.200 ip and 8888 port.
user2’s JupyterLab container bind to the 172.18.0.201 ip and 8888 port.

user1 can access to his/her container using the user1.alpine-docker-host subdomain.
user2 can access to his/her container using the user2.alpine-docker-host subdomain.

## User authentication

The default.conf file defines two Nginx “server” section for the user’s subdomains: user1.alpine-docker-host and user2.alpine-docker-host

The server sections forwards the traffic to the appropriate backend server (proxy_pass) and forces user authentication (auth_basic_user_file).

Each user has an own password file in the htpasswd directory:
* user1: htpasswd/user1
* user2. htpasswd/user2

**Always change these default passwords to your own strong passwords !**

These files contains a username:password pair. In the sample htpasswd files the password is user1 for the user1 user and user2 for the user2 user.

To create a new password file, use the following command in the htpasswd directory. The command will prompt for a password:

```
htpasswd -c ./user3 user3
```

## SSL Certificates

If you have a signed wildcard certificate, you can simply copy it into the ssl-certs/key.pem and ssl-certs/cert.pem files.

To generate self-signed certificates, use the `cert-gen.sh` command in the ssl-certs directory.

## How to run the container

If the current directory is /mnt/docker-persistent-volumes/

Clone this repository:

```
git clone https://github.com/gergelyszerovay/docker-nginx-reverse-proxy-for-jupyterlab 
```

Generate self-signed certificate:

```
cd ssl-certs
sh ./cert-gen.sh
```

Set up the users in default.conf and their password files in the htpasswd folder, then run the container (the container is based on the official Nginx image):

```
docker run -d --restart always \
  --network=host \
  --name nginx-reverse-proxy-for-jupyterlab \
  -v /mnt/docker-persistent-volumes/nginx-reverse-proxy-for-jupyterlab/default.conf:/etc/nginx/conf.d/default.conf:ro \
  -v /mnt/docker-persistent-volumes/nginx-reverse-proxy-for-jupyterlab/htpasswd:/etc/nginx/htpasswd:ro \
  -v /mnt/docker-persistent-volumes/nginx-reverse-proxy-for-jupyterlab/ssl-certs:/ssl-certs:ro \
  nginx
```

To get the configuration errors, enter:

```
docker logs nginx-reverse-proxy-for-jupyterlab
```

If there is no output, it means there is no errors.

## How to add/remove users

To add a new user (for example: userX), assign a new subdomain (userX.alpine-docker-host) to the Docker host’s ip. Create a new password file in the htpasswd directory:

```
htpasswd -c ./userX userX
```

And finally append the following to the end of default.conf:

```
# userX
server {
    listen 443 ssl;
    server_name userX.alpine-docker-host;

    ssl_certificate /ssl-certs/cert.pem;
    ssl_certificate_key /ssl-certs/key.pem;

    gzip on;
    gzip_types text/plain text/css text/js text/xml text/javascript application/javascript application/x-javascript application/json application/xml application/rss+xml image/svg+xml;

    location / {
        proxy_pass https://172.18.0.X:8888;
        auth_basic_user_file /etc/nginx/htpasswd/userX;

        auth_basic "Username and password:";

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;

        proxy_http_version 1.1;
        proxy_redirect off;
        proxy_buffering off;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
}
```

Modify the proxy_pass, auth_basic_user_file and server_name settings according to the username and the ip address of the user’s JupyterLab container.

To remove an user, delete its “server” part from default.conf and remove its password file form the htaccess directory.

