# PKI workshop

This workshop will simulate the scenario in how we can use PKI to make secure communication. All this session we work on terminal, you can following in step-by-step.

## Prerequisite

- OpenSSL
- cURL
- Docker
- tree

## Use self-sign to secure communication

Scenario: We will provide simple web server and map SSL certificate on it. We create connection with cURL and debug it to see what's happen in handshake state. Then, we compare between insecure and secure.  

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

6. You can use `cURL` to see what happen.

    Get web page with insecure protocol(HTTP).

    ```bash
    $ curl -v http://localhost:8080

    *   Trying 127.0.0.1:8080...
    * Connected to localhost (127.0.0.1) port 8080 (#0)
    > GET / HTTP/1.1
    > Host: localhost:8080
    > User-Agent: curl/7.74.0
    > Accept: */*
    > 
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 200 OK
    < Server: nginx/1.20.2
    < Date: Mon, 18 Apr 2022 11:45:35 GMT
    < Content-Type: text/html
    < Content-Length: 612
    < Last-Modified: Tue, 16 Nov 2021 15:04:23 GMT
    < Connection: keep-alive
    < ETag: "6193c877-264"
    < Accept-Ranges: bytes
    < 
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>
 
    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>
 
    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    * Connection #0 to host localhost left intact
    ```

    Use `curl` in default option and see **error**

    ```bash
    $ curl -v https://localhost:8443

    *   Trying 127.0.0.1:8443...
    * Connected to localhost (127.0.0.1) port 8443 (#0)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * successfully set certificate verify locations:
    *  CAfile: /etc/ssl/certs/ca-certificates.crt
    *  CApath: /etc/ssl/certs
    * TLSv1.3 (OUT), TLS handshake, Client hello (1):
    * TLSv1.3 (IN), TLS handshake, Server hello (2):
    * TLSv1.2 (IN), TLS handshake, Certificate (11):
    * TLSv1.2 (OUT), TLS alert, unknown CA (560):
    * SSL certificate problem: self signed certificate
    * Closing connection 0
    curl: (60) SSL certificate problem: self signed certificate
    More details here: https://curl.se/docs/sslcerts.html

    curl failed to verify the legitimacy of the server and therefore could not
    establish a secure connection to it. To learn more about this situation and
    how to fix it, please visit the web page mentioned above.
    ```

    In the error response, We can translate in a part of message.  

    1. They're try to make a connection to endpoint.
       >\*   Trying 127.0.0.1:8443...  
       >\* Connected to localhost (127.0.0.1) port 8443 (#0)

    2. They're prepared information for connection.
       >\* ALPN, offering h2  
       >\* ALPN, offering http/1.1  
       >\* successfully set certificate verify locations:  
       >\*  CAfile: /etc/ssl/certs/ca-certificates.crt  
       >\*  CApath: /etc/ssl/certs  

    3. They're send first contact message in `hello`.
       >\* TLSv1.3 (OUT), TLS handshake, Client hello (1):  
       >\* TLSv1.3 (IN), TLS handshake, Server hello (2):

    4. Server send digital certificate to client.
       >\* TLSv1.2 (IN), TLS handshake, Certificate (11):

    5. Client verify certificate integrity and found in self-sign issue.
       >\* TLSv1.2 (OUT), TLS alert, unknown CA (560):
       >\* SSL certificate problem: self signed certificate

    6. Client close connection and report error message.
       >\* Closing connection 0  
       > curl: (60) SSL certificate problem: self signed certificate  
       > More details here: 'https://curl.se/docs/sslcerts.html'  
       >  
       > curl failed to verify the legitimacy of the server and therefore could not  
       > establish a secure connection to it. To learn more about this situation and  
       > how to fix it, please visit the web page mentioned above.  

7. You can use `crt` certificate file in previous step to trust this communication.

    You can see normal establish connection in debug log.

    ```bash
    $ curl -v --cacert ssl/ssl-lab.example.local.crt https://localhost:8443

    *   Trying 127.0.0.1:8443...
    * Connected to localhost (127.0.0.1) port 8443 (#0)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * successfully set certificate verify locations:
    *  CAfile: ssl/ssl-lab.example.local.crt
    *  CApath: /etc/ssl/certs
    * TLSv1.3 (OUT), TLS handshake, Client hello (1):
    * TLSv1.3 (IN), TLS handshake, Server hello (2):
    * TLSv1.2 (IN), TLS handshake, Certificate (11):
    * TLSv1.2 (IN), TLS handshake, Server key exchange (12):
    * TLSv1.2 (IN), TLS handshake, Server finished (14):
    * TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
    * TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
    * TLSv1.2 (OUT), TLS handshake, Finished (20):
    * TLSv1.2 (IN), TLS handshake, Finished (20):
    * SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
    * ALPN, server accepted to use http/1.1
    * Server certificate:
    *  subject: C=TH; ST=BKK; O=Opsta; OU=DevOps; CN=ssl-lab.example.local
    *  start date: Apr 17 15:41:24 2022 GMT
    *  expire date: Apr 20 15:41:24 2022 GMT
    *  subjectAltName: host "localhost" matched cert\'s "localhost"
    *  issuer: C=TH; ST=BKK; O=Opsta; OU=DevOps; CN=ssl-lab.example.local
    *  SSL certificate verify ok.
    > GET / HTTP/1.1
    > Host: localhost:8443
    > User-Agent: curl/7.74.0
    > Accept: */*
    > 
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 200 OK
    < Server: nginx/1.20.2
    < Date: Mon, 18 Apr 2022 12:03:50 GMT
    < Content-Type: text/html
    < Content-Length: 612
    < Last-Modified: Tue, 16 Nov 2021 15:04:23 GMT
    < Connection: keep-alive
    < ETag: "6193c877-264"
    < Accept-Ranges: bytes
    ```
