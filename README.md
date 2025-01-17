# The Things Stack LoRaWAN Network Server

![The Things Stack CE for Raspberry Pi](https://raw.githubusercontent.com/xoseperez/the-things-stack-docker/master/resources/logo_small.png)

This project deploys the The Things Stack LoRaWAN Network Server (Open Source Edition) on a Raspberry Pi or equivalent SBC using docker.

The initial script performs a series of tasks, prior to boot the service. These tasks are:

* Build a configuration file based on environment variables
* Create a self signed certificate
* Configure the identity database
  * Initialize it
  * Create an admin
  * Create oauth clients for the CLI and the console

This is a Work In Progress. It should work just fine for local (LAN) deployments, still needs a lot of testing for other environments.

## Requirements

### Hardware

* Raspberry Pi 3/4 or [balenaFin](https://www.balena.io/fin/)
* SD card in case of the RPi 3/4
* Power supply and (optionally) ethernet cable

### Software

* An image of [Raspberry Pi OS](https://www.raspberrypi.org/software/operating-systems/) or [Ubuntu Server for ARM](https://ubuntu.com/download/raspberry-pi).
* Docker and docker-compose (see instruction below)

### Network

Even thou they are not necessary, you may want to have:

* a static IP for the device (either in the device itself or using a DHCP lease on your router)
* a domain (or a subdomain of a domain you already have) pointing to the device

Check the `Configuring the domain` section below for different options to fulfull these two options.


## Deploy

You can use the next `docker-compose.yml` file to configure and run your instance of Basics™ Station. 

```
version: '3.7'

volumes:
    redis: 
    postgres:
    stack-blob:
    stack-data:

services:

  postgres:
    image: postgres:11.12
    container_name: postgres
    restart: unless-stopped
    environment:
        - POSTGRES_PASSWORD=root
        - POSTGRES_USER=root
        - POSTGRES_DB=ttn_lorawan
    volumes:
        - 'postgres:/var/lib/postgresql/data'
    ports:
        - "127.0.0.1:5432:5432"
    
  redis:
    image: redis:6.2.4-alpine3.13
    container_name: redis
    command: redis-server --appendonly yes
    restart: unless-stopped
    volumes:
        - 'redis:/data'
    ports:
        - "127.0.0.1:6379:6379"
  
  stack:
    image: xoseperez/the-things-stack:latest
    container_name: stack
    restart: unless-stopped
    depends_on:
        - redis
        - postgres
    volumes:
        - 'stack-blob:/srv/ttn-lorawan/public/blob'
        - 'stack-data:/srv/data'
    environment:
        TTS_DOMAIN: 192.168.42.1        # set this to the IP or domain name of the host you will be using to access the stack
        TTN_LW_BLOB_LOCAL_DIRECTORY: /srv/ttn-lorawan/public/blob
        TTN_LW_REDIS_ADDRESS: redis:6379
        TTN_LW_IS_DATABASE_URI: postgres://root:root@postgres:5432/ttn_lorawan?sslmode=disable

    ports:
    
        - "80:1885"
        - "443:8885"
    
        - "1881:1881"
        - "1882:1882"
        - "1883:1883"
        - "1884:1884"
        - "1885:1885"
        - "1887:1887"
    
        - "8881:8881"
        - "8882:8882"
        - "8883:8883"
        - "8884:8884"
        - "8885:8885"
        - "8887:8887"
    
        - "1700:1700/udp"
  ```

Modify the `TTS_DOMAIN` environment variables to match your setup. 

## Log in

Point your browser to the first local IP of the device or to the domain name (if you have defined one) using HTTPS and use the default credentials (admin/changeme) to log in as administrator.

## Details

### Resetting values

Certificates are recreated if TTS_DOMAIN or any TTS_SUBJECT_* variable below changes.
Database is reset if TTS_DOMAIN, TTS_ADMIN_EMAIL, TTS_ADMIN_PASSWORD or TTS_CONSOLE_SECRET change.

### Configuring the IP and domain

You want to assign your device a fixed IP or a domain name where you can always reach it. Here you have a few clues on how to do it.

#### Static IP

To reach the IP or to properly configure a domain or subdomain you will have to configure the Raspberry Pi with a static address. You have two options here:

1. Configure a static lease on your home router linking the RPi MAC with an IP. Everytime the RPi boots it will ask for an IP using DHCP (this is the default) and router will allways gfive it the same IP.

2. Configure a static IP on the RPi itself instead of using DHCP. 

#### Configuring the domain or subdomain

Once you know the PI will always be accessible at the same IP, there are a number of ways to define a domain name or a subdomain pointing to the device IP. After doing any of these approaches change the TTS_DOMAIN environment variable accordingly so the stack service recreates the right certificates for the domain.

1. Using a DNS in your LAN, like PiHole, dnsmask,... these will work great inside your LAN. But this option requires an extra step since BalenaOS by default uses Google DNS servers (8.8.8.8). So you have to instruct it to use your local DNS server instead. 

2. Using a third party service, like Cloudflare, for instance. If you are managing a domain from such a service you can just add an A register for a subdomain pointing to your local (or public) IP address.

```
A lns.ttn.cat 192.168.1.25
```

Then you just have to wait for the domain name to propagate.

### Variables

Variable Name | Value | Description | Default
------------ | ------------- | ------------- | -------------
**TTS_SERVER_NAME** | `STRING` | Name of the server | The Things Stack
**TTS_DOMAIN** | `STRING` | Domain | Required, will not boot if undefined
**TTS_ADMIN_EMAIL** | `STRING` | Admin email | admin@thethings.example.com
**TTS_NOREPLY_EMAIL** | `STRING` | Email used for communications | noreply@thethings.example.com
**TTS_ADMIN_PASSWORD** | `STRING` | Admin password (change it here or in the admin profile) | changeme
**TTS_CONSOLE_SECRET** | `STRING` | Console secret | console
**TTS_DEVICE_CLAIMING_SECRET** | `STRING` | Device claiming secret | device_claiming
**TTS_METRICS_PASSWORD** | `STRING` | Metrics password | metrics
**TTS_PPROF_PASSWORD** | `STRING` | Profiling password | pprof
**TTS_SMTP_HOST** | `STRING` | SMTP Server |  
**TTS_SMTP_USER** | `STRING` | SMTP User |  
**TTS_SMTP_PASS** | `STRING` | SMTP Password |  
**TTS_SENDGRID_KEY** | `STRING` | Sendgrid API Key (SMTP_HOST has to be empty in order to use this) | 
**TTS_SUBJECT_COUNTRY** | `STRING` | Self Certificate country code| ES
**TTS_SUBJECT_STATE** | `STRING` | Self Certificate state | Catalunya
**TTS_SUBJECT_LOCATION** | `STRING` | Self Certificate city | Barcelona
**TTS_SUBJECT_ORGANIZATION** | `STRING` | Self Certificate organization | TTN Catalunya

## Troubleshooting

* If you are having certificates problems or "token rejected" message on the TTS website, try regenerating the credentials by changing any of the SUBJECT_* variables. You can also open a terminal to the `stack` service, delete the `/srv/data/certificates_signature` file and restart the stack service.

* If the database fails to initialize the best way to force the start script to init it again is to change any of these variables: TTS_ADMIN_EMAIL, TTS_ADMIN_PASSWORD or TTS_CONSOLE_SECRET. You can also open a terminal to the `stack` service, delete the `/srv/data/database_signature` file and restart the stack service.

* When the database is reconfigured (because you change any of the environment variables in the previous point) the passwords for the admin and the console are overwritten. So if you are logged in as admin you will have to logout and login again with the default password.

## TODO

* Lots of testing :)
* Testing performance (# of devices) on different platforms
* Option to use ACME / Let's Encrypt for valid certificates
* Option to configure a connection to the Packet Broker

## Attribution

- This is based on the [The Things Network LoRaWAN Stack repository](https://github.com/TheThingsNetwork/lorawan-stack).
- This is in joint effort by [Xose Pérez](https://twitter.com/xoseperez/) and [Marc Pous](https://twitter.com/gy4nt/) from the TTN community in Barcelona.
