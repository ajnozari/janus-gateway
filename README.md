# Janus Gateway Guide

Setup janus gateway on docker behind an nginx reverse proxy

## Getting Started

First edit the files in Janus-CFG to the config you require. You can also mount them as a volume so they can be edited separately
from the image itself.

Depending on the plugins you have activated you may need to edit the following:

janus.jcfg  
janus.plugin.videoroom.jcfg  
janus.transport.http.jcfg  
janus.transport.websockets.jcfg  

These files MUST be edited to enable http rest interface (janus.transport.http.jcfg), as well as other features that are needed. It is reccomended to enable the rest API along with a secret for it, so you can manage
the server through the Restful API that is made available.
Please note these secrets are not considered secure as they will need to be provided to any client that wishes to connect, with the exception of the admin secret which is only used to edit the server in realtime.

## Build Docker Image:
Next up is building the docker image.

`docker build -t ajnozari/janus-gateway-docker .` 

this should build an updated image with the right dependencies installed, credit goes to @atyenoria for 
building this dockerfile, I simply made edits to fix small issues I was having, most notably with the 
http post api.

At this point you can run the docker image and expose the ports. However if you continue on you can setup nginx to 
handle SSL through lets-encrypt, and use WSS and HTTPS instead of open connections.

You can now start the Janus server with the following docker command:

`docker run --net=host --name="janus" -it -t ajnozari/janus-gateway-docker >> /dev/null`

this will launch janus and put it's output into the background.  Additionally you can launch this through portainer.

if this does not work, you can try this alternative that uses the docker bridge, but note it will cause a minor performance hit.

`docker run -p 8088:8088 -p 8188:8188 -p 7088:7088 --name="janus" -it -t ajnozari/janus-gateway-docker >> /dev/null`

## Install Nginx:
I have installed nginx on the host, i don't feel the need to dockerize it, especially since we are using nginx to handle SSL. You can always substitute your own ssl cert at this point. However I will not cover setting up a manual certificate.

First make sure your repo is up to date
`$ sudo apt update && sudo apt upgrade`

Then install nginx from ubuntu's repo
`$ sudo apt install nginx`

## Firewall:
(optional)  
If you have setup ubuntu's firewall, you should check that nginx is allowed

List the applications with:
`sudo ufw app list`

It will output something like:
```
 Available applications:
   Nginx Full
   Nginx HTTP
   Nginx HTTPS
   OpenSSH
```

enable nginx with 
```bash
sudo ufw allow 'Nginx HTTP'
sudo ufw allow 'Nginx HTTPS'
```

The firewall may also be inactive, if this is the case you can and probably should enable it.

You can verify the change by typing:

`sudo ufw status`

## Nginx Site Conf
Next you need to delete the default file in:

/etc/nginx/sites-available and 
/etc/nginx/sites-enabled

Then copy the janus.conf file from this repo to /etc/nginx/sites-available.
Be sure to edit it to match your server's url as well as any other specific changes you might need. For most people what I have provided will work well with let's encrypt.

Then run the following to enable the site:
`sudo ln -s /etc/nginx/sites-avaible/janus.conf /etc/nginx/sites-enabled`

Then check the config to make sure no formatting errors have occured
`sudo nginx -t`

If everything returns ok apply and restart nginx:
`sudo systemctl restart nginx`

Now navigate to http://<yourjanusurl>/janus/info

If you see the janus info page then you know that you've successfully setup janus and nginx

## Let's Encrypt
Next up is let's encrypt, which lets us simplify the nginx ssl config.

First Add Certbot PPA to the list of repositories
```bash
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
```

Next Install Certbot:
```bash
sudo apt-get install certbot python-certbot-nginx
```

Now, request a certificate:

`sudo certbot --nginx`

Follow the prompts and when prompted allow certbot to make the necessary changes to your nginx conf for ssl.  Be sure to restart nginx once everything is completed.

Once you have accomplished this task, test the certbot renewal with the following, it should show success if it passed the first time.

`sudo certbot renew --dry-run`

Now you can test with the same endpoint as before but with HTTPS instead:

https://<yourjanusurl>/janus/info

If everything is setup correctly you should see that your site is secured.

## Janus Endpoints

This guide provides three endpoints you can access.
https://<janusurl>/janus - the default http endpoint for restful api and long-polling clients 
wss://<janusurl>/janus-ws - the default websocket endpoint for clients
https://<janusurl>/admin - the default janus admin endpoint

The default HTTP/s endpoint for the restful api is useful for managing janus through a backend site or service. The full documentation for janus's rest-api can be found on meetecho's website. 
Additionally, you can provide an array to the server instead of a single url ['wss://<janusurl>/janus-ws', 'https://<janusurl>/janus']. If for whatever reason you cannot make a connection over websockets, 
janus will default to long polling.
 