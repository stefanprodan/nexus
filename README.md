# Nexus Repository Manager OSS

[![](https://images.microbadger.com/badges/image/stefanprodan/nexus.svg)](https://microbadger.com/images/stefanprodan/nexus "Get your own image badge on microbadger.com")

A Sonatype Nexus Repository Manager image based on Alpine with OpenJDK 8.

Running Nexus with:
* Repository on port 8081
* Docker Registry (hosted) on port 5000
* Data persistence on a host directory

```
docker run -d --name nexus \
    -v /path/to/nexus-data:/nexus-data \
	-p 8081:8081 \
	-p 5000:5000 \
	stefanprodan/nexus
```

## Notes

* Default credentials are: `admin` / `admin123`

* It can take some time for the service to launch in a new container. You can tail the log to determine once Nexus is ready:

```
$ docker logs -f nexus
```

* Test with curl:

```
$ curl -u admin:admin123 http://localhost:8081/service/metrics/ping
```

* Installation of Nexus is to `/opt/sonatype/nexus-version`.  

* A persistent directory, `/nexus-data`, is used for configuration, logs, and storage.

* Three environment variables can be used to control the JVM arguments

  * `JAVA_MAX_MEM`, passed as -Xmx.  Defaults to `1200m`.

  * `JAVA_MIN_MEM`, passed as -Xms.  Defaults to `1200m`.

  * `EXTRA_JAVA_OPTS`.  Additional options can be passed to the JVM via this variable.


## Configure NGINX as reverse proxy for Nexus and hosted Docker Registry

***nginx.conf***

```
worker_processes 1;

events { 
	worker_connections 1024; 
}

http {
	error_log /var/log/nginx/error.log warn;
	access_log  /dev/null;
	proxy_intercept_errors off;
	proxy_send_timeout 120;
	proxy_read_timeout 300;
	
	upstream nexus {
        server nexus:8081;
	}

	upstream registry {
        server nexus:5000;
	}

	server {
        listen 80;
        server_name nexus.demo.com;

        keepalive_timeout  5 5;
        proxy_buffering    off;

        # allow large uploads
        client_max_body_size 1G;

        location / {
		# redirect to docker registry
		if ($http_user_agent ~ docker ) {
			proxy_pass http://registry;
		}
		proxy_pass http://nexus;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto "https";
        }
    }
}
``` 

Replace ***nexus.demo.com*** with your own domain. The NGINX server detects if a call is made by the docker client, based on user agent, and redirects that call to the Docker Registry. You alose need to setup a certificate with letsencrypt for Docker Registry to work over SSL.

## Nexus and Nginx setup

```
docker network create nexus-net

docker run -d --name nexus \
    -v /path/to/nexus-data:/nexus-data \
    --restart unless-stopped \
    --network nexus-net \
	stefanprodan/nexus
	
docker run -d -p 80:80 -p 443:443 --name nginx \
    -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
    --restart unless-stopped \
    --network nexus-net \
    nginx
```

## Changelog

* 24 Dec 2016  - Nexus 3.2.0-01
* 19 Sept 2016 - Nexus 3.0.2-02
* 25 Feb 2017 - Nexus 3.2.1-01



