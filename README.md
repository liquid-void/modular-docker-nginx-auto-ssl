# Automated SSL certificates in a modular docker-compose setup
Multi-module docker-compose example project to showcase the powerful combination of [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) with [acme-companion](https://github.com/nginx-proxy/acme-companion) and docker networks for:

- **Auto-detection** of docker services.
- **Auto-generation** of nginx configurations and restarting nginx. 
- **Automated creation, renewal and use** of letsencrypt SSL certificates.
- Storing of certificates and configurations in docker volumes independently from the host system for an added layer of safety.

## Introduction
This simple modular approach sets up **nginx-proxy** (including dockergen) and **acme-companion** together in a separate compose file module, here called proxy. Service compositions can be brought up or down easily whenever necessary, since they are uncoupled from the proxy tools. Communication between service modules and proxy is realized via **importing the proxy docker network** in a new docker-compose file: 

```yaml
networks:
  front:
    external:
      name: proxy_front
```
This allows for automated complex service compositions in a production environment without having to mess with nginx configurations or certificate generation yourself.

This method works with most images out-of-the-box. To highlight possible use-cases, this example includes the following popular database management and graphing solutions:

- [InfluxDB v2.0.x](https://github.com/influxdata/influxdb), which now comes bundled with Chronograph client in one image
- [Grafana](https://github.com/grafana/grafana), which connects to InfluxDB via importing influxdb_back network  
- [MongoDB](https://github.com/mongodb/mongo) with [mongo-express](https://github.com/mongo-express/mongo-express) client



Environment variables for example service:

```
VIRTUAL_HOST=grafana.yourdomain.tld      # allows nginx-proxy to generate server blocks config
VIRTUAL_PORT=3000                        # only needed for ports other than 80
LETSENCRYPT_HOST=grafana.yourdomain.tld
LETSENCRYPT_EMAIL=info@yourdomain.tld    # only needed when email differs from default email
```

# Get started

### **Step 1: Launch nginx-proxy & acme-companion**
Let's first bring up our proxy module:
```sh
cd proxy
docker-compose up -d
```
nginx/dockergen and acme-companion are now both listening for new docker services. You should see some similar log excerpts:

<details>
  <summary>nginx-proxy & nginx-proxy-acme logs</summary>
  
  ```sh
  user@host:~/modular-docker-nginx-auto-ssl/proxy$ docker logs nginx-proxy -f
dockergen.1 | 2021/10/02 12:45:29 Generated '/etc/nginx/conf.d/default.conf' from 2 containers
dockergen.1 | 2021/10/02 12:45:29 Running 'nginx -s reload'
nginx.1     | 2021/10/02 12:45:29 [notice] 31#31: signal 1 (SIGHUP) received from 43, reconfiguring
nginx.1     | 2021/10/02 12:45:29 [notice] 31#31: reconfiguring
dockergen.1 | 2021/10/02 12:45:29 Watching docker events
^C

user@host:~/modular-docker-nginx-auto-ssl/proxy$ docker logs nginx-proxy-acme -f
2021/10/02 12:45:30 Generated '/app/letsencrypt_service_data' from 2 containers
2021/10/02 12:45:30 Running '/app/signal_le_service'
2021/10/02 12:45:30 Watching docker events
2021/10/02 12:45:30 Contents of /app/letsencrypt_service_data did not change. Skipping notification '/app/signal_le_service'
[Sat Oct  2 12:45:31 UTC 2021] Create account key ok.
[Sat Oct  2 12:45:31 UTC 2021] Registering account: https://acme-v02.api.letsencrypt.org/directory
[Sat Oct  2 12:45:32 UTC 2021] Registered
[Sat Oct  2 12:45:32 UTC 2021] ACCOUNT_THUMBPRINT='zhxdF8?????????REDACTED????????????'
Reloading nginx proxy (5a66246?????????????????REDACTED???????????????????????????)...
  ```
  
</details>

<br/>

### **Step 2: Bring up the first service module**
Next, let's do the same for influxdb module:
```sh
cd ../services/influxdb
docker-compose up -d
```
Check the logs of nginx-proxy:
```sh
docker logs nginx-proxy -f
```
<details>
  <summary>nginx-proxy log output</summary>
  
  ```sh
dockergen.1 | 2021/10/02 13:11:59 Received event start for container 403abbbf078c
dockergen.1 | 2021/10/02 13:11:59 Generated '/etc/nginx/conf.d/default.conf' from 3 containers
dockergen.1 | 2021/10/02 13:11:59 Running 'nginx -s reload'
nginx.1     | 2021/10/02 13:11:59 [notice] 31#31: signal 1 (SIGHUP) received from 151, reconfiguring
nginx.1     | 2021/10/02 13:11:59 [notice] 31#31: reconfiguring
nginx.1     | 2021/10/02 13:11:59 [notice] 31#31: using the "epoll" event method
nginx.1     | 2021/10/02 13:11:59 [notice] 31#31: start worker processes
nginx.1     | 2021/10/02 13:11:59 [notice] 31#31: start worker process 152
nginx.1     | 2021/10/02 13:11:59 [notice] 31#31: start worker process 153
nginx.1     | 2021/10/02 13:11:59 [notice] 31#31: start worker process 154

...

nginx.1     | influx.example.com 3.xxx.xxx.xx - - [02/Oct/2021:13:12:11 +0000] "GET /.well-known/acme-challenge/FtmV5BAXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX HTTP/1.1" 200 87 "-" "Mozilla/5.0 (compatible; Let's Encrypt validation server; +https://www.letsencrypt.org)" "-"
nginx.1     | influx.example.com 3.xxx.xxx.xx - - [02/Oct/2021:13:12:12 +0000] "GET /.well-known/acme-challenge/FtmV5BAXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX HTTP/1.1" 200 87 "-" "Mozilla/5.0 (compatible; Let's Encrypt validation server; +https://www.letsencrypt.org)" "-"
nginx.1     | influx.example.com 64.xxx.xxx.xx - - [02/Oct/2021:13:12:12 +0000] "GET /.well-known/acme-challenge/FtmV5XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX HTTP/1.1" 200 87 "-" "Mozilla/5.0 (compatible; Let's Encrypt validation server; +https://www.letsencrypt.org)" "-"
nginx.1     | influx.example.com 52.xxx.xxx.xx - - [02/Oct/2021:13:12:12 +0000] "GET /.well-known/acme-challenge/FtmV5BAXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX HTTP/1.1" 200 87 "-" "Mozilla/5.0 (compatible; Let's Encrypt validation server; +https://www.letsencrypt.org)" "-"
  ```
  
</details>

<br/>

We can see that nginx-proxy/dockergen has detected influxdb service and generated its configuration. Then the acme-challenge is executed.


Next, check the logs of nginx-proxy-acme:
```sh
docker logs nginx-proxy-acme -f
```
<details>
  <summary>nginx-proxy-acme log output</summary>
  
  ```sh
2021/10/02 13:11:59 Received event start for container 403abbbf078c
2021/10/02 13:12:04 Debounce minTimer fired
2021/10/02 13:12:04 Generated '/app/letsencrypt_service_data' from 3 containers
2021/10/02 13:12:04 Running '/app/signal_le_service'
Creating/renewal influx.example.com certificates... (influx.example.com)
[Sat Oct  2 13:12:07 UTC 2021] Using CA: https://acme-v02.api.letsencrypt.org/directory
[Sat Oct  2 13:12:07 UTC 2021] Creating domain key
[Sat Oct  2 13:12:08 UTC 2021] The domain key is here: /etc/acme.sh/info@example.com/influx.example.com/influx.example.com.key
[Sat Oct  2 13:12:08 UTC 2021] Single domain='influx.example.com'
[Sat Oct  2 13:12:08 UTC 2021] Getting domain auth token for each domain
[Sat Oct  2 13:12:11 UTC 2021] Getting webroot for domain='influx.example.com'
[Sat Oct  2 13:12:11 UTC 2021] Verifying: influx.example.com
[Sat Oct  2 13:12:14 UTC 2021] Success
[Sat Oct  2 13:12:14 UTC 2021] Verify finished, start to sign.
[Sat Oct  2 13:12:14 UTC 2021] Lets finalize the order.
[Sat Oct  2 13:12:14 UTC 2021] Le_OrderFinalize='https://acme-v02.api.letsencrypt.org/acme/finalize/XXXXXXXX/XXXXXXXX'
[Sat Oct  2 13:12:15 UTC 2021] Downloading cert.
[Sat Oct  2 13:12:15 UTC 2021] Le_LinkCert='https://acme-v02.api.letsencrypt.org/acme/cert/03a5ceXXXXXXXXXXXXXXXXXXXXXXXX'
[Sat Oct  2 13:12:16 UTC 2021] Cert success.
-----BEGIN CERTIFICATE-----
MIIGKDCCBRCgAwIBAgISA6XO4RwF6xr+elO9s99wQRW9MA0GCSqGSIb3DQEBCwUA
..
..
..
.. REDACTED
..
..
..
Y0GkoTg2OB1qLgefDKd3CchyW/Mw0l/gNyi88uX/orp0v4mJPPzCWsFDb0M=
-----END CERTIFICATE-----
[Sat Oct  2 13:12:16 UTC 2021] Your cert is in  /etc/acme.sh/info@example.com/influx.example.com/influx.example.com.cer
[Sat Oct  2 13:12:16 UTC 2021] Your cert key is in  /etc/acme.sh/info@example.com/influx.example.com/influx.example.com.key
[Sat Oct  2 13:12:16 UTC 2021] The intermediate CA cert is in  /etc/acme.sh/info@example.com/influx.example.com/ca.cer
[Sat Oct  2 13:12:16 UTC 2021] And the full chain certs is there:  /etc/acme.sh/info@example.com/influx.example.com/fullchain.cer
[Sat Oct  2 13:12:16 UTC 2021] Installing cert to:/etc/nginx/certs/influx.example.com/cert.pem
[Sat Oct  2 13:12:16 UTC 2021] Installing CA to:/etc/nginx/certs/influx.example.com/chain.pem
[Sat Oct  2 13:12:16 UTC 2021] Installing key to:/etc/nginx/certs/influx.example.com/key.pem
[Sat Oct  2 13:12:16 UTC 2021] Installing full chain to:/etc/nginx/certs/influx.example.com/fullchain.pem
Reloading nginx proxy (5a66246774038f40XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX)...
  ```
  
</details>

<br/>






Ideally, the output tells us, that acme-companion generated a new letsencrypt certificate for indluxdb.yourdomain.tld.

Open your browser, and navigate to: https://indluxdb.yourdomain.tld. You should be greeted with an onboarding page for InfluxDB UI, formerly known as Chronograph.
You can now setup an initial user, organisation and bucket.


<br/>

### **Step 3: Bring up another service**
Next, let's repeat these steps for Grafana.
```sh
cd ../grafana
docker-compose up -d
```
The resulting logs will be similar to the ones shown in step 2.

The difference here is, that in the Grafana module, we're not only importing proxy_front network, but also influxdb_back.
```yaml
networks:
  front:
    external:
      name: proxy_front
  back:
    external:
      name: influxdb_back
```

This allows the grafana service to connect to InfluxDB, giving us more graphing tools for the same data. 
Grafana should now be reachable under the following link: grafana.yourdomain.tld
You will be greeted with a login page. Login using the default credentials:
```
user: admin
pass: admin
```
After that you'll be prompted to set a new password.

You can now connect to InfluxDB via Configuration-> Data Sources -> InfluxDB.
There are only 3 settings required:
- url: http://influxdb:8086  # make use of docker service naming
- organisation: your org name set during InfluxDB setup
- token: your token that got generated automatically, after setting up the first user. Find it in InfluxDB -> Data -> Tokens

<br/>

### **Step 4: Want more?**

In the last example, you can bring up MongoDB and mongo-express UI in a similar fahsion.
Mongo-express database explorer should become available under mongo.yourdomain.tld.

<br/>

## Other tested images, that work out-of-the-box
- [Jenkins](https://github.com/jenkinsci/docker)
- [Gogs](https://github.com/gogs/gogs)


<br/>

# What's next?
- Setup messaging service, to inform administrator about newly issued certificates or issues
- Include a custom example UI built from source
- Make docker.sock readable for InfluxDB Telegraf to collect performance metrics


<br/>

# Troubleshooting
- Make sure your host is publicly reachable on both port 80 and 443. Check your firewall rules.
- Make sure you have a valid A record set up for all subdomains, so they can be resolved in the acme challenges.
- Every service needs to expose the port it is listening on inside the container (not the host system) in order for nginx-proxy and acme-companion to be able to detect it. This can be achieved either in the build setup by using EXPOSE 'port_number' in Dockerfile or post-build with the 
  ```-expose "port_number"``` parameter in docker-compose.yml. This won't expose ports to the host system, but make them available to services on the same docker network. In our example this can be omitted, since we're just pulling images from Dockerhub, which already have their ports exposed in their respective Dockerfiles.
- VIRTUAL_PORT is needed for every service, that is listening on a different port than the default, port 80.

For more details, refer to the documentation of:
- nginx-proxy: https://github.com/nginx-proxy/nginx-proxy
- acme-companion: https://github.com/nginx-proxy/acme-companion