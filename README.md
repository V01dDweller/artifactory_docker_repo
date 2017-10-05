# How to Create a Private Docker Registry in Artifactory Pro #

Below are the stages required to create a private Docker registry using Artifactory Pro 5.x with an nginx-based reverse-proxy. The final stage is the configuration needed for a docker client to use the new registry. This requires Artifactory Pro (5 or greater) as Artifactory OSS does not allow creating Docker repos.

**Stages**
1. Create a Docker Repo in Artifactory
2. Create an OpenSSL Wildcard Certificate
3. Create a Nginx Reverse-Proxy Config File
4. Install Nginx
5. Configure Nginx as a Reverse-Proxy
6. Configure a Docker Client to Access the Private Registry

---

### Stage 1: Create a Docker Repo in Artifactory ###

Create a simple Docker V2 repo in Artifactory Pro, accepting the defaults. Perform these steps in the Artifactory Pro web gui.

1. Log into the Artifactory Pro web-gui using the Admin account

2. Navigate to local repos

    ```bash
    Admin -> Repositories -> Local
    ```

3. Click "(+) New" to create a new local repo
4. From the available "Package Types" choose "docker"
5. Enter a "Repository Key"

   **Note:** This will become the repository name at the end of the Artifactory repo URL, e.g.
   
        http://artifactoryhost.mydomain.com/artifactory/docker-repo-name

        *... and the corresponding docker registry sub-domain*

        https://docker-repo-name.artifactoryhost.mydomain.com/image:tag

6. Accept all defaults and click the **Save & Finish** button

---

### Stage 2. Create OpenSSL Wildcard Certificate ###

While a CA-issued SSL cert would be ideal, this step is for creating a self-signed one. These steps should be executed on the Artifactory Pro host via the Linux command line.

1. Create the OpenSSL self-signed certificate and key files:

  ```bash
  sudo openssl req\
  -x509\
  -nodes -days 3650\
  -newkey rsa:2048\
  -keyout /etc/ssl/private/nginx-selfsigned.key\
  -out /etc/ssl/certs/nginx-selfsigned.crt
  ```

2. Create a Diffie-Hellman group

  ```bash
  sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
  ```

---

### Stage 3. Create a Nginx Reverse-proxy Config File ###

Generate a reverse-proxy config for nginx using the Artifactory Pro Proxy Configuration Generator and the SSL files created in Stage 2.

1. Log into the Artifactory Pro web gui as an Admin.

2. Navigate to:

   ```bash
   Admin -> Configuration -> Reverse Proxy
   ```
3. Fill out the "Reverse Proxy Configuration" form:

   Server Provider: `nginx` (drop-down)

   Internal Hostname\*: `artifactoryhost.mydomain.com`

   Internal Port\*: `8081` (default)

   Internal Context Path: `artifactory` (default)

   [x] Use HTTP

   HTTP Port \*: `80`

   [x] Use HTTPS

   HTTPS Port \*: `443`

   SSL Key Path \*

   ```bash
   /etc/ssl/private/nginx-selfsigned.key

   ```

   SSL Certificate Path \*

   ```bash
   /etc/ssl/certs/nginx-selfsigned.crt
   ```

   *Docker Reverse Proxy Settings*

   Reverse Proxy Method: `Sub Domain`

   Server Name Expression: `*.artifactoryhost.mydomain.com`

4. Click the "Save" button

5. Click "Download" (upper left corner) to get the `artifactory.conf` file for nginx.

---

### Stage 4. Install Nginx ###

**Note:** This is for RHEL7, repos for other Linux distributions can be found in the [nginx wiki](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)

1. Install the nginx repo

2. Create /etc/yum.repos.d/nginx.repo

   ```apache
   [nginx]
   name=nginx repo
   baseurl=https://nginx.org/packages/rhel/7/x86_64/
   gpgcheck=0
   enabled=1
   proxy=http://proxy.mydomain.com:8080
   ```

3. Install nginx

   ```bash
   yum install nginx
   ```

---

### Stage 5. Configure Nginx as a Reverse-Proxy ###

1. Back up/delete the nginx default config:

   ```bash
   sudo cp -v /etc/nginx/conf.d/{default.conf,default.conf.orig}
   ```

2. Copy the `artifactory.conf` file downloaded in Stage 3 to /etc/nginx/conf.d/ on the Artifactory host:

   Sample `artifactory.conf`

   ```nginx
   ###########################################################
   ## this configuration was generated by JFrog Artifactory ##
   ###########################################################

   ## add ssl entries when https has been set in config
   ssl_certificate      /etc/ssl/certs/nginx-selfsigned.crt;
   ssl_certificate_key  /etc/ssl/private/nginx-selfsigned.key;
   ssl_session_cache shared:SSL:1m;
   ssl_prefer_server_ciphers   on;
   ## server configuration
   server {
       listen 443 ssl;
       listen 80 ;
       server_name ~(?<repo>.+)\.artifactoryhost.mydomain.com artifactoryhost.mydomain.com;

       if ($http_x_forwarded_proto = '') {
           set $http_x_forwarded_proto  $scheme;
       }
       ## Application specific logs
       ## access_log /var/log/nginx/artifactoryhost.mydomain.com-access.log timing;
       ## error_log /var/log/nginx/artifactoryhost.mydomain.com-error.log;
       rewrite ^/$ /artifactory/webapp/ redirect;
       rewrite ^/artifactory/?(/webapp)?$ /artifactory/webapp/ redirect;
       rewrite ^/(v1|v2)/(.*) /artifactory/api/docker/$repo/$1/$2;
       chunked_transfer_encoding on;
       client_max_body_size 0;
       location /artifactory/ {
       proxy_read_timeout  900;
       proxy_pass_header   Server;
       proxy_cookie_path   ~*^/.* /;
       if ( $request_uri ~ ^/artifactory/(.*)$ ) {
           proxy_pass          http://artifactoryhost.mydomain.com:8081/artifactory/$1;
       }
       proxy_pass          http://artifactoryhost.mydomain.com:8081/artifactory/;
       proxy_set_header    X-Artifactory-Override-Base-Url $http_x_forwarded_proto://$host:$server_port/artifactory;
       proxy_set_header    X-Forwarded-Port  $server_port;
       proxy_set_header    X-Forwarded-Proto $http_x_forwarded_proto;
       proxy_set_header    Host              $http_host;
       proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
       }
   }
   ```

4. Create a new file -  /etc/nginx/conf.d/ssl.conf

   ```nginx
   server {
       listen 443 http2 ssl;
       listen [::]:443 http2 ssl;
   
       server_name server_IP_address;
   
       ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
       ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
       ssl_dhparam /etc/ssl/certs/dhparam.pem;
   }
   ```

5. Restart nginx

   ```bash
   systemctl restart nginx
   ```

---

### Stage 6. Configure a Docker Client to Access the Private Registry ###

1. Securely transfer the `nginx-selfsigned.crt` file to the Docker client host

2. Copy it to `/etc/pki/ca-trust/source/anchors/` with a new name:

   RHEL7 example ([other examples](https://docs.docker.com/registry/insecure/#docker-still-complains-about-the-certificate-when-using-authentication)):

   ```bash
   sudo cp -v nginx-selfsigned.crt /etc/pki/ca-trust/source/anchors/docker-repo-name.artifactoryhost.mydomain.com.crt
   ```

3. Update CA Trust

   ```bash
   sudo update-ca-trust
   ```

4. Add the new docker registry subdomain to /etc/hosts (or ask your network admin to update DNS):

   ```bash
   sudo echo "10.10.10.10  docker-repo-name.artifactoryhost.mydomain.com.crt" >> /etc/hosts
   ```

5. Optional: Add your new private docker registry to docker's proxy excludes

   ```apache
   [Service]
   Environment="HTTP_PROXY=http://proxy.mydomain.com:8080/" "HTTPS_PROXY=http://proxy.mydomain.com:8080/"  "NO_PROXY=localhost,127.0.0.1,*.mydomain.com,*.mydomain.com,docker-repo-name.artifactoryhost.mydomain.com"
   ```
6.  Reload systemd daemon configs:

   ```
   systemctl deamon-reload
   ```

 7. Restart docker:

   ```
   systemctl restart docker
   ```

---

### Testing the new Docker Registry ###
Test the new docker repo via a pull or push from the  docker client:

```bash
docker push docker-repo-name.artifactoryhost.mydomain.com/myimage:tag
```
