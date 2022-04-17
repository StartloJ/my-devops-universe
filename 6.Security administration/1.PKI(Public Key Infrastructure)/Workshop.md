# PKI workshop

This workshop will simulate the scenario in how we can use PKI to make secure communication. All this session we work on terminal, you can following in step-by-step.

## Prerequisite

- OpenSSL
- cURL
- Docker
- tree

## Let do it

Scenario, we will provide simple web server and map SSL certificate on it. We create connection with cURL and debug it to see what's happen in handshake state. Then, we compare between insecure and secure.  

1. Generate SSL with CN is `ssl-lab.example.local`.

   ```bash
    $ openssl req -newkey rsa:4096 -nodes -keyout ssl/ssl-lab.example.local.key \
        -x509 -sha256 -days 3 \
        -subj "/C=TH/ST=BKK/O=Opsta/OU=DevOps/CN=ssl-lab.example.local" \
        -addext "subjectAltName = DNS:ssl-lab.example.local, DNS:localhost, DNS:127.0.0.1" \
        -out ssl/ssl-lab.example.local.crt

    Generating a RSA private key
    ...................++++
    ..............................................................................++++
    writing new private key to 'ssl-lab.example.local.key'
    -----
   ```

   Check file should created.

   ```bash
    $ tree ssl

    ssl
    ├── ssl-lab.example.local.crt
    └── ssl-lab.example.local.key
    0 directories, 2 files
   ```

2. Verify certificate with openssl

    ```bash
    $ openssl x509 -noout -modulus -in ssl/ssl-lab.example.local.crt | openssl md5
    (stdin)= ac9173e222ab6b766e49da3069268dd0
    # OUTPUT will difference in your terminal

    $ openssl rsa -noout -modulus -in ssl/ssl-lab.example.local.key | openssl md5
    (stdin)= ac9173e222ab6b766e49da3069268dd0
    ```

    If your see same result in 2 command, you're right.

3. Now, you already to create web server in docker for learn how SSL work. First, you will check your docker daemon working and run a sample Nginx web server.

    Create a Dockerfile in following.

    ```Dockerfile
    FROM nginx:1.20-alpine

    ADD config/default.conf /etc/nginx/conf.d/default.conf
 
    COPY ssl /etc/nginx/ssl
 
    RUN chown -R 0:0 /etc/nginx/ssl \
        && chown -R 0:0 /etc/nginx/conf.d/default.conf
    ```

    Create Nginx configure in `config/`

    ```ini
    server {
        listen       80;
        server_name  localhost ssl-lab.example.local;
 
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
 
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
 
    server {
        listen              443 ssl;
        server_name         localhost ssl-lab.example.local;
        keepalive_timeout   70;
 
        ssl_certificate     /etc/nginx/ssl/ssl-lab.example.local.crt;
        ssl_certificate_key /etc/nginx/ssl/ssl-lab.example.local.key;
        ssl_protocols       TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers off;
 
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
 
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
    ```

    Then, build customize http server in docker.

    ```bash
    $ docker build -t opsta/http:lab .

    Sending build context to Docker daemon  233.5kB
    Step 1/4 : FROM nginx:1.20-alpine
     ---> 5c05ca045835
    Step 2/4 : ADD config/default.conf /etc/nginx/conf.d/default.conf
     ---> Using cache
     ---> 1284fcf47f5b
    Step 3/4 : COPY ssl /etc/nginx/ssl
     ---> 920c31460cbf
    Step 4/4 : RUN chown -R 0:0 /etc/nginx/ssl     && chown -R 0:0 /etc/nginx/conf.d/default.conf
     ---> Running in f8bfa038944f
    Removing intermediate container f8bfa038944f
     ---> 725ee42e058b
    Successfully built 725ee42e058b
    Successfully tagged opsta/http:lab
    ```

4. You can run http server in terminal.

    ```bash
    docker run -it --rm -p 8080:80 -p 8443:443 opsta/http:lab
    ```

5. Verify http server has running with `docker ps` or `curl http://localhost:8080`. It just return result on STDOUT.

    ```bash
    $ docker ps

    CONTAINER ID   IMAGE            COMMAND                  CREATED         STATUS         PORTS                                                                            NAMES
    1235d24fe059   opsta/http:lab   "/docker-entrypoint.…"   7 minutes ago   Up 7 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp, 0.0.0.0:8443->443/tcp, :::8443->443/tcp   relaxed_montalcini
    ```

6. 