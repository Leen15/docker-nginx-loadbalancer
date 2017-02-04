# docker-nginx-loadbalancer

This image will auto-generate its own config file for a load-balancer.
It's a fork of jasonwyatt/docker-nginx-loadbalancer but also support dynamic dns link and custom access log format.


It looks for environment variables in the following formats:

    <service-name>_<service-instance-id>_PORT_80_TCP_ADDR=x.x.x.x
    <service-name>_PATH=<some path>

Optional/Conditional environment variables:

    <service-name>_REMOTE_PORT=<remoteport> (optional - default: 80)
    <service-name>_REMOTE_PATH=<remotepath> (optional - default: /)
    <service-name>_BALANCING_TYPE=[ip_hash|least_conn] (optional)
    <service-name>_EXPOSE_PROTOCOL=[http|https|both] (optional - default: http)
    <service-name>_HOSTNAME=<vhostname> (required if <service-name>_EXPOSE_PROTOCOL is https or both)
    <service-name>_ACCESS_LOG=[/dev/stdout|off] (optional - default: /dev/stdout)
    <service-name>_ERROR_LOG=[/dev/stdout|/dev/null] (optional - default: /dev/stdout)
    <service-name>_LOG_LEVEL=[emerg|alert|crit|error|warn|notice|info|debug'] (optional - default: error)
    <service-name>_LOG_MODE=upstreamlog (optional - default: blank)
    <service-name>_LOG_FORMAT=upstreamlog + compatible log format (optional - default: blank)
    <env-formatted-vhostname>_SSL_CERTIFICATE=<something.pem> (required if the vhost will need ssl support)
    <env-formatted-vhostname>_SSL_CERTIFICATE_KEY=<something.key> (required if the vhost will need ssl support)
    <env-formatted-vhostname>_SSL_DHPARAM=<dhparam.pem> (required if the vhost will need ssl support)
    <env-formatted-vhostname>_SSL_CIPHERS=<"colon separated ciphers wrapped in quotes"> (required if the vhost will need ssl support)
    <env-formatted-vhostname>_SSL_PROTOCOLS=<protocol (e.g. TLSv1.2)> (required if the vhost will need ssl support)

**ATTENTION: Nginx has a problem with DNS in upstream: if you use DNS (and not IP) in ADDR envs, nginx will resolve to the ip ONLY AT THE STARTUP, so if then these dns change their IP, NGINX will not work anymore.**   
For avoid this, I added two more ENVS:     
   
    <service-name>_LINK_SERVICE=<web> (optional - default: blank)   
    <service-name>_RESOLVER_ADDR=<169.254.169.250> (optional - default: blank)   
   
With these two envs, you can attach a link to the container and this dns will be used and updated automatically.   
With rancher, you can link a service and round robin balacing works fine.   
Resolver_addr it's the dns server that should be used to resolve the dns (check in /etc/resolv.conf inside your container)   

**I also added the option to ban clients with specific User Agent or IP.**
**Nginx will check for remote_addr and http_x_forwarded_for, so the ban works also if this nginx is after another proxy (that sends X-Forwarded-For header):**
 
    <service-name>_BLOCK_USER_AGENT=<trident|chrome> (optional - default: blank)   
    <service-name>_BLOCK_IP=<192.168.0.1,192.168.0.1,bad_ip> (optional - default: blank)   
Keep attention:   
- User agent strings are case insensitive and should be concatenated with a pipe ("|")   
- IP ban doesn't work with partial IP or with CIDR address notation. It works with single ips concatenated with a comma (",").    

Example 1:

    # automatically created environment variables (docker links)
    WEBAPP_LINK_SERVICE=web
    WEBAPP_1_PORT_80_TCP_ADDR=web
    WEBAPP_RESOLVER_ADDR=169.254.169.250
    WEBAPP_LOG_LEVEL=error
    WEBAPP_PATH=/
    WEBAPP_EXPOSE_PROTOCOL=http
    WEBAPP_ACCESS_LOG=/dev/stdout
    WEBAPP_ERROR_LOG=/dev/stdout
    WEBAPP_ACCESS_LOG_MODE=upstreamlog
    WEBAPP_ACCESS_LOG_FORMAT=upstreamlog '[$time_local] - fwd=$remote_addr container=$upstream_addr response_status=$status status=$upstream_status path="$request" response=$upstream_response_time total_time=$request_time bytes=$body_bytes_sent user_agent="$http_user_agent" host=$http_host body=$request_body'

Generates (/etc/nginx/sites-enabled/proxy.conf):

    upstream webappstream {
        server web:80;
    }
    
    map $remote_addr $denied_remote {
      default 0;
    }
    
    map $http_x_forwarded_for $denied_forwarded {
      default 0;
    }
    
    log_format upstreamlog '[$time_local] - fwd=$remote_addr container=$upstream_addr response_status=$status status=$upstream_status path="$request" response=$upstream_response_time total_time=$request_time bytes=$body_bytes_sent user_agent="$http_user_agent" host=$http_host body=$request_body';
            

    server {
        listen 80;

        server_name 0.0.0.0;

        error_log /dev/stdout error;
        access_log /dev/stdout upstreamlog;

        root /usr/share/nginx/html;

        # For service: WEBAPP
        location / {
            resolver 169.254.169.250 valid=5s;
            resolver_timeout 5s;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Real-IP $remote_addr;
            set $webapp web;
            proxy_pass http://$webapp;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_buffering off;
        }
    }
And logs like this in stdout:

      [11/Oct/2016:16:06:08 +0000] - fwd=10.42.24.138 container=10.42.96.50:80 response_status=200 status=200 path="GET / HTTP/1.1" response=0.002 total_time=0.002 bytes=0 user_agent="Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.116 Safari/537.36" host=www.example.com body=-
      [11/Oct/2016:16:08:30 +0000] - fwd=10.42.24.138 container=10.42.171.179:80 response_status=200 status=200 path="GET /ping HTTP/1.1" response=0.387 total_time=0.387 bytes=7106 user_agent="Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.116 Safari/537.36" host=www.example.com body=-

   
   
   
Example 2:

    # automatically created environment variables (docker links)
    WEBAPP_1_PORT_80_TCP_ADDR=192.168.0.2
    WEBAPP_2_PORT_80_TCP_ADDR=192.168.0.3
    WEBAPP_3_PORT_80_TCP_ADDR=192.168.0.4
    API_1_PORT_80_TCP_ADDR=192.168.0.5
    API_2_PORT_80_TCP_ADDR=192.168.0.6
    TOMCAT_1_PORT_8080_TCP_ADDR=192.168.0.7
    TOMCAT_2_PORT_8080_TCP_ADDR=192.168.0.8

    # special environment variables
    WEBAPP_PATH=/
    WEBAPP_BALANCING_TYPE=ip_hash
    WEBAPP_EXPOSE_PROTOCOL=both
    WEBAPP_HOSTNAME=www.example.com
    WEBAPP_ACCESS_LOG=off
    WEBAPP_ERROR_LOG=/dev/stdout
    WEBAPP_LOG_LEVEL=emerg
    API_PATH=/api/
    API_EXPOSE_PROTOCOL=https
    API_HOSTNAME=www.example.com
    WWW_EXAMPLE_COM_SSL_CERTIFICATE=ssl/something.pem
    WWW_EXAMPLE_COM_SSL_CERTIFICATE_KEY=ssl/something.key
    WWW_EXAMPLE_COM_SSL_DHPARAM=ssl/dhparam.pem
    WWW_EXAMPLE_COM_SSL_CIPHERS="ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256"
    WWW_EXAMPLE_COM_SSL_PROTOCOLS=TLSv1.2
    TOMCAT_PATH=/javaapp
    TOMCAT_REMOTE_PORT=8080
    TOMCAT_REMOTE_PATH=/javaapp

Generates (/etc/nginx/sites-enabled/proxy.conf):

    upstream webapp {
        ip_hash;
        server 192.168.0.2;    
        server 192.168.0.3;    
        server 192.168.0.4;    
    }

    upstream api {
        server 192.168.0.5;
        server 192.168.0.6;
    }

    upstream tomcat {
        server 192.168.0.7;
        server 192.168.0.8;
    }

    server {
        listen 80;
        listen [::]:80 ipv6only=on;
        server_name www.example.com;

        error_log /dev/stdout emerg;
        access_log off;

        root /usr/share/nginx/html;

        location / {
            proxy_pass http://webapp:80/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_buffering off;
        }
    }

    server {
        listen 443;
        server_name www.example.com;

        root html;
        index index.html index.htm;

        ssl on;
        ssl_certificate ssl/something.pem;
        ssl_certificate_key ssl/something.key;
        
        # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
        ssl_dhparam ssl/dhparam.pem;

        ssl_session_timeout 5m;

        ssl_protocols TLSv1.2;
        ssl_ciphers "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256";
        ssl_prefer_server_ciphers on;

        root /usr/share/nginx/html;

        location / {
            proxy_pass http://webapp:80/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_buffering off;
        }
        location /api/ {
            proxy_pass http://api:80/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_buffering off;
        }
    }

    server {
        listen 80;
        listen [::]:80 ipv6only=on;

        root /usr/share/nginx/html;

        location /javaapp {
            proxy_pass http://tomcat:8080/javaapp;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_buffering off;
        }
    }
