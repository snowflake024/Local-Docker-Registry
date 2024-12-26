# How to set up this private Docker registry.

Private registries contain proprietary or otherwise sensitive container images not intended for public distribution. By requiring authentication, these registries restrict who can pull and push images, creating a more secure development environment. By running locally, private registries also speed up app development and save bandwidth.

## Install and Configure Private Docker Registry

Setting up a server to host a private Docker registry requires running a registry service, configuring an Nginx web server, and generating the necessary security certificates. Follow the steps below to install a private Docker registry on a server.
### Step 1: Create Registry Directories

Create a new project directory to store all the required configuration files. 

1. Use the following mkdir command to create a new project directory labeled registry and two sub-directories, nginx and auth:

```bash
mkdir -p /registry/{nginx,auth}
```

2. Create two new directories, conf.d and ssl, inside the nginx directory:

```bash
mkdir -p registry/nginx/{conf.d,ssl}
```

3. Go to the registry directory and inspect the directory hierarchy by using the tree command:

```bash
cd registry && tree

.
├── README.md
├── auth
│   └── registry.passwd
├── cert
│   ├── ca_reg.
│   ├── ca_reg.crt
│   ├── ca_reg.key
│   ├── ca_reg.srl
│   ├── ctreg.big-brain.local.crt
│   ├── ctreg.big-brain.local.csr
│   ├── ctreg.big-brain.local.key
│   └── v3.ext
├── docker-compose.yaml
└── nginx
    ├── conf.d
    │   ├── additional.conf
    │   └── registry.conf
    └── ssl
        ├── fullchain.pem
        └── privkey.pem
```

The output shows the final structure of the registry project directory.

### Step 2: Create Docker Compose Manifest and Define Services

1. Create a new compose.yaml manifest for Docker Compose.

```bash
vim docker-compose.yaml
```

2. Paste the following content into the file:

```yaml
version: '3'
services:
  registry:
    image: registry:2
    restart: always
    ports:
    - "5000:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry-Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.passwd
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
    volumes:
      - registrydata:/data
      - ./auth:/auth
    networks:
      - registrynet
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d/:/etc/nginx/conf.d/
      - ./nginx/ssl/:/etc/nginx/ssl/
    networks:
      - registrynet
networks:
  registrynet:
    driver: bridge
volumes:
  registrydata:
    driver: local
```

The first service defined in the manifest is the registry service. The other service runs an Nginx web server. The configuration sets the service to run on ports 80:80 (HTTPS) and 443:443 (HTTPS). It mounts the local directories for virtual configuration (conf.d) and SSL certificates (ssl). Finally, the manifest defines Docker networks and volumes used by the registry. The registrynet network runs with the bridge driver, while the registrydata volume uses the local driver.

3. Save and close the file.

### Step 3: Set up Nginx Port Forwarding

Follow the procedure below to configure a Nginx server block by defining the needed parameters in a configuration file.

1. Create a new server block configuration file named registry.conf in the nginx/conf.d directory:

```bash
vim  nginx/conf.d/registry.conf
```

2. Add the following content, replacing [domain] with the registry's domain address, e.g., example.com:

```nginx
upstream docker-registry {
    server registry:5000;
}

server {
    listen 80;
    server_name [domain];
    return 301 https://[domain]$request_uri;
}

server {
    listen 443 ssl http2;
    server_name registry.[domain];

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;

    location / {
        if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" )  {
            return 404;
        }

        proxy_pass                          http://docker-registry;
        proxy_set_header  Host              $http_host;
        proxy_set_header  X-Real-IP         $remote_addr;
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Proto $scheme;
        proxy_read_timeout                  900;
    }

}
```

3. Save the file and exit.

The code above defines the following components:

    A connection between Nginx and the registry service via port 5000.
    A server block that listens on port 80 and performs HTTP to HTTPS redirection.
    A server block that listens on port 443 and provides an encrypted connection for the registry. The block is also configured to allow connections only from Docker clients with version 1.6 or later because earlier versions mishandled user agents.

Step 4: Increase Nginx File Upload Size

By default, Nginx limits the file upload size to 1MB. As many Docker images exceed this number, the best practice is to increase the maximum file size on Nginx. To ensure the successful upload of larger images, follow the steps below to increase the limit to 2GB:

1. Create an additional Nginx configuration file by typing:

```bash
vim  nginx/conf.d/additional.conf
```

2. Add the following line in the file:

```nginx
client_max_body_size 2G;
```

3. Save and close the file.
   
### Step 5: Configure/Generate SSL Certificate and Basic Authentication

Note: You can have a self-signed certificate for the registry or have a CA sign a valid public certificate.

#### Generate a Certificate Authority Certificate

In a production environment, you should obtain a certificate from a CA. In a test or development environment, you can generate your own CA. To generate a CA certficate, run the following commands.

Generate a CA certificate private key.
```
openssl genrsa -out ca.key 4096
```

Generate the CA certificate. Adapt the values in the -subj option to reflect your organization. If you use an FQDN to connect your local registry, you must specify it as the common name (CN) attribute.
```
openssl req -x509 -new -nodes -sha512 -days 3650 \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=MyPersonal Root CA" \
    -key ca.key \
    -out ca.crt
```

Generate a Server Certificate. The certificate usually contains a .crt file and a .key file, for example, yourdomain.com.crt and yourdomain.com.key.

Generate a private key.
```
openssl genrsa -out yourdomain.com.key 4096
```

Generate a certificate signing request (CSR). Adapt the values in the -subj option to reflect your organization. If you use an FQDN to connect your local registry host, you must specify it as the common name (CN) attribute and use it in the key and CSR filenames.
```
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
    -key yourdomain.com.key \
    -out yourdomain.com.csr
```
    
Generate an x509 v3 extension file. Regardless of whether you’re using either an FQDN or an IP address to connect to your local registry host, you must create this file so that you can generate a certificate for your Harbor host that complies with the Subject Alternative Name (SAN) and x509 v3 extension requirements. Replace the DNS entries to reflect your domain.
```
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=yourdomain.com
DNS.2=yourdomain
DNS.3=hostname
EOF
```

Use the v3.ext file to generate a certificate for your local registry host.

Replace the yourdomain.com in the CSR and CRT file names with the local registry host name.
```
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in yourdomain.com.csr \
    -out yourdomain.com.crt
```

Proceed with the below steps to copy the domain's SSL certificates to the ssl directory in the project:

1. Use the cp command to copy the fullchain.pem file:

```bash
cp /registry/cert/yourdomain.com.crt nginx/ssl/fullchain.pem
```

2. Copy the privkey.pem file:

```bash
cp /registry/cert/yourdomain.com.key nginx/ssl/privkey.pem
```

3. Go to the auth directory:

```bash
cd auth
```

4. request a new password file named registry.passwd for the user:

Note: htpasswd binary is inckuded in httpd-tools/apache2-tools package

```bash
htpasswd -Bc registry.passwd [username]
```

5. Type in a strong password and re-type it to confirm. The output confirms the success of the operation.
Requesting a password for a user with htpasswd.

### Step 6: Add Root CA Certificate to the system store (if it's self signed)

1. Copy the ca certifiacate over to the store and run the system utility to include the CA in the OS CA bundle
```
cp /path/to/cert/ca.crt /etc/pki/ca-trust/source/anchors/ # ( or cp /path/to/cert/ca.crt /usr/share/ca-certificates/extra/ )
update-ca-trust # ( or dpkg-reconfigure ca-certificates for debian based )
```

2. Verify if the previous steps have made the CA trusted
```
openssl verify -CAfile /etc/ssl/certs/ca-bundle.crt /path/to/cert/ca.crt
```
If the output is anything different from OK then further troubleshooting should ensue or steps should be repeated.

### Step 7: Add certificate to docker (Optional)
Add the Root CA certificate to Docker and the host system by following the procedure below:

1. Create a directory for Docker certificates:
```bash
mkdir -p /etc/docker/certs.d/[domain]
```

2. Copy the Root certificate into the directory:
```bash
cp /path/to/cert/ca.crt /etc/docker/certs.d/[domain]
```

3. Restart the Docker service to apply the changes:
```bash
systemctl restart docker
```

Step 7: Run Docker Registry

With everything set up and ready, build the Docker Registry and Nginx containers using Docker Compose:

1. Execute the docker compose up command with the -d option to deploy the containers in the detached mode.

```bash
docker compose up -d
```

Starting the Docker registry with Docker Compose.

2. Check if the registry and the nginx services are running:

```bash
docker compose ps
```

The output should show the services and their assigned ports.
How to Push Docker Image to Private Registry

1. To push an image from a Docker host to the private Docker registry server, log in to the registry with the following command:

```bash
docker login https://registry.[domain]/v2/
```

For example, to access the registry at example.com, type:

```bash
docker login https://registry.example.com/v2/
```

2. Type in the username and password created in Step 5.
Logging in to a private Docker registry.

3. Push the image to the private registry with the command:

```bash
docker push registry.[domain]/[new-image-name]
```

Pull Image from Docker Hub to Private Registry

1. To locally store an image from Docker Hub to a private registry, use the docker pull command:

```bash
docker pull [image]
```

2. Add a tag to the image to label it for the private registry:

```bash
docker image tag [image] registry.[domain]/[new-image-name]
```

3. Check whether the Docker image is locally available by prompting the system to list all locally stored images:

```bash
docker images
```

Conclusion
