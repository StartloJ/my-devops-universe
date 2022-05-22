# PKI workshop

This workshop will simulate the scenario in how we can use PKI to make secure communication. All this session we work on terminal, you can following in step-by-step.

In the late session, we learned about how to use public/private keys in an easy case.  
I explained the workshop to implement a solution. Then, We upgrade to an advanced workshop  
to understand how to implement public/private CA intermediate servers.  
If you are not read previous topic, you can go to:

- [what-we-should-know-in-pki](https://dev.to/startpher/what-we-should-know-in-pki-l25)
- [little-step-to-use-pki-easiest](https://dev.to/startpher/little-step-to-use-pki-easiest-mmg)

## Prerequisite

- OpenSSL
- cURL
- Docker
- tree

## Create root CA certificate

In first thing, we will generate simple self-sign root CA certificate with following step:  

1. Create SSL private key

   ```bash
   $ openssl genrsa -out rootCAkey.pem 4096
   Generating RSA private key, 4096 bit long modulus (2 primes)
   .........................................++++
   .........++++
   e is 65537 (0x010001)
   ```

   > We generate random keys with 4,096 character and you can use other lengths like 1024, 2048, 8192, or ETC. You can increase or decrease your key length to optimize cpu power to process them.

2. Create Certificate from private key

   ```bash
   $ openssl req -x509 -sha256 -new -nodes -key rootCAkey.pem \
      -days 14 -out rootCAcert.pem
   You are about to be asked to enter information that will be incorporated
   into your certificate request.
   What you are about to enter is what is called a Distinguished Name or a DN.
   There are quite a few fields but you can leave some blank
   For some fields there will be a default value,
   If you enter '.', the field will be left blank.
   -----
   Country Name (2 letter code) [AU]:TH
   State or Province Name (full name) [Some-State]:Bangkok
   Locality Name (eg, city) []:BangKhea
   Organization Name (eg, company) [Internet Widgits Pty Ltd]:Opsta Thailand
   Organizational Unit Name (eg, section) []:DevOps
   Common Name (e.g. server FQDN or YOUR name) []:OpstaRootCA
   Email Address []:watcharin@opsta.co.th
   ```

   > For the previous command, I request a new certificate file with some information. I use the digest algorithm `SHA-256` and set it to expire before **14 days**. It shows a prompt on your terminal for adding more information. You can see an example above.

3. Now, we can verify own root CA

   ```bash
   openssl x509 -text -in rootCAcert.pem
   ```

   > You can see your certificate information. Then go to the next step to create a chain certificate.

## Create server certificate with issuer

1. Generate random private key with same step root CA

   ```bash
   # You can replace rsaServerKey.pem to your name
   # And can change key size from 2048 to other length
   $ openssl genrsa -out rsaServerKey.pem 2048
   Generating RSA private key, 2048 bit long modulus (2 primes)
   ......................................................................+++++
   ...............+++++
   e is 65537 (0x010001)
   ```

2. Create CNF file that provide meta data for CSR

   ```ini
   # req.cnf
   [req]
   req_extensions = v3_req
   distinguished_name = dn
   prompt = no

   [dn]
   CN = ssl-lab.example.local
   C = TH
   L = Bangkok
   O = Opsta Thailand
   OU = DevOps

   [v3_req]
   subjectAltName = @san_names

   [san_names]
   DNS.1 = ssl-lab.example.local
   DNS.2 = localhost
   DNS.3 = 127.0.0.1
   ```

   > Provide simple configuration to create CSR file. This file has **SAN(Subject Alternative Name)** with 3 names in `[san_names]`. For `@san_names`, I use for extend alternative name for server.

3. Generate CSR certificate

   ```bash
   $ openssl req -new -out req.csr -key rsaServerKey.pem -sha256 -config req.cnf
   $ cat req.csr
   -----BEGIN CERTIFICATE REQUEST-----
   MIIC9zCCAd8CAQAwaTEeMBwGA1UEAwwVc3NsLWxhYi5leGFtcGxlLmxvY2FsMQsw
   CQYDVQQGEwJUSDEQMA4GA1UEBwwHQmFuZ2tvazEXMBUGA1UECgwOT3BzdGEgVGhh
   aWxhbmQxDzANBgNVBAsMBkRldk9wczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCC
   …
   k9jkEKhPeICqJgHLbyiEcEc9xaPTMPb35cBQT8irnUq1+WbQamhWDIBmwzDHCSyf
   jCK672ACRleQCj8kk5l+dOB0wzWEaDcCoQNwIwqg2RpfDKc1lHARroBzm1P/4grg
   NaF5EYJJWhXUXGKq68meHpCTGzgC7M06rwBdOR+8l0GIUZ5K25MvNg3+ZA==
   -----END CERTIFICATE REQUEST-----
   ```

4. Verify CSR information, we focus on Requested Extension.

   ```bash
   $ openssl req -in req.csr -noout -text
   …
   Requested Extensions:
               X509v3 Subject Alternative Name: 
                   DNS:ssl-lab.example.local, DNS:localhost, DNS:127.0.0.1
   ```

5. Create trusted certificate with root CA with CSR file

   ```bash
   $ openssl x509 -req -sha256 -in req.csr -CA ../rootCAcert.pem \ 
       -CAkey ../rootCAkey.pem -CAcreateserial -out CertServer.pem -days 7
   …
   Signature ok
   subject=CN = ssl-lab.example.local, C = TH, L = Bangkok, O = Opsta Thailand, OU = DevOps
   Getting CA Private Key
   ```

   > We create trusted certificates with digest algorithm `SHA-256` and create root CA serials that track how many certificates were issued with `-CAcreateserial` , so if *not the first certificate* you will use `-CAserial` instead. Then, this certificate will stay for 7 days only.

6. Verify certificate information that should have issuer

   ```bash
   $ openssl x509 -text -in CertServer.pem
   …
   Issuer: C = TH, ST = Bangkok, L = BangKhea, O = Opsta Thailand, OU = DevOps, CN = OpstaRootCA, emailAddress = watcharin@opsta.co.th
   ```

   > We will see Issuer information under Certificate >> Data.

## Conclusion

Now you can replace this certificate and private key with the previous session (also topic name [little-step-to-use-pki-easiest](./Workshop-selfsign.md)) and test again. You will install your custom root CA to your browser or machine to see a difference between a single self-signed certificate and certificate with CA issuer.

Back to first page.  
[Go to README](./README.md)
