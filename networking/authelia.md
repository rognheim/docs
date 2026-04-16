---
title: Authelia
last_reviewed: 2026-03-07
owner: Rognheim
---

# :simple-webauthn:


## **Authelia**

---

### **Introduction**

Authelia is a powerful open-source authentication server designed to fortify self-hosted applications with advanced security features. Offering single sign-on (SSO) and multi-factor authentication (MFA) capabilities, Authelia streamlines user access while maintaining a high level of protection for your public facing applications. Authelia integrates seamlessly with popular reverse proxies and is able to orchestrate secure access across multiple applications, providing a centralized, user-friendly authentication experience.

![Authelia Overview](https://www.authelia.com/overview/prologue/architecture/architecture-diagram.png)

---

### **Configuration files**

<details>
<summary>docker-compose.yml</summary>

```yaml
---
# Authelia

secrets:
  jwt_secret:
    file: ./config/secrets/jwt_secret # password should be 64 or more characters
  encryption_key:
    file: ./config/secrets/encryption_key # password should be 64 or more characters
  session_secret:
    file: ./config/secrets/session_secret # password should be 64 or more characters
  smtp_password:
    file: ./config/secrets/smtp_password

services:
  authelia:
    image: authelia/authelia
    container_name: authelia
    restart: unless-stopped
    healthcheck:
      disable: true
    secrets:
      - jwt_secret
      - encryption_key
      - session_secret
      - smtp_password
    environment:
      - TZ=Europe/Oslo
      - AUTHELIA_JWT_SECRET_FILE=/config/secrets/jwt_secret
      - AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE=/config/secrets/encryption_key
      - AUTHELIA_SESSION_SECRET_FILE=/config/secrets/session_secret
      - AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE=/config/secrets/smtp_password
    volumes:
      - ./config:/config
    networks:
      - public
    labels:
      - traefik.enable=true
      - traefik.http.routers.authelia.entrypoints=https
      - traefik.http.routers.authelia.rule=Host(`auth.example.com`)
      - traefik.http.routers.authelia.tls=true
      - traefik.http.routers.authelia.service=authelia
      - traefik.http.services.authelia.loadbalancer.server.port=9091
      - traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.example.com/
      - traefik.http.middlewares.authelia.forwardauth.trustForwardHeader=true
      - traefik.http.middlewares.authelia.forwardauth.authResponseHeaders=Remote-User, Remote-Groups, Remote-Name, Remote-Email
      - diun.enable=true

networks:
    public:
        external: true

```
</details>

<details>
<summary>configuration.yml</summary>

```yaml
---
# Authelia Configuration

server:
  host: 0.0.0.0
  port: 9091
log:
  level: info
  # file_path: "/var/log/authelia_access.log"
  keep_stdout: true
theme: dark
# jwt_secret: AUTHELIA_JWT_SECRET_FILE
default_redirection_url: https://auth.example.com
totp:
  issuer: authelia.com

# duo_api:
#  hostname: api-123456789.example.com
#  integration_key: ABCDEF
#  # This secret can also be set using the env variables AUTHELIA_DUO_API_SECRET_KEY_FILE
#  secret_key: 1234567890abcdefghifjkl

ntp:
  address: "time.cloudflare.com:123"
  version: 3
  max_desync: 3s
  disable_startup_check: true
  disable_failure: false

authentication_backend:
  file:
    path: /config/user_database.yml
    password:
      algorithm: argon2id
      iterations: 1
      parallelism: 8
      salt_length: 16
      memory: 64

access_control:
  default_policy: deny
  rules:
    # Rules applied to everyone

    # bypass:
#    - domain: example.com
#      policy: bypass
    # one_factor:
    - domain: speedtest.example.com
      policy: one_factor
    # two_factor:
    - domain: traefik.example.com
      policy: two_factor
    - domain: code.example.com
      policy: two_factor

session:
  name: authelia_session
  # This secret can also be set using the env variables AUTHELIA_SESSION_SECRET_FILE
  # secret: AUTHELIA_SESSION_SECRET_FILE
  expiration: 3600  # 1 hour
  inactivity: 300  # 5 minutes
  domain: example.com  # Should match whatever your root protected domain is

  # redis:
  #   host: redis
  #   port: 6379
  #   # This secret can also be set using the env variables AUTHELIA_SESSION_REDIS_PASSWORD_FILE
  #   # password: authelia

regulation:
  max_retries: 3
  find_time: 120
  ban_time: 300

storage:
  local:
    path: /config/db.sqlite3
  # encryption_key: AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE

notifier:
  smtp:
    username: username@gmail.com
    # This secret can also be set using the env variables AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE
    # password: password
    host: smtp.gmail.com
    port: 587
    sender: "Authelia <admin@example.no>"
    subject: "[Authelia] {title}"
  # filesystem:
  #   filename: /config/notification.txt
```
</details>

<details>
<summary>user_database.yml</summary>

```yaml
users:
  user:
    disabled: false
    displayname: "User"
    password: "$argon2id$v=19$m=65536,t=3,p=2$BpLnfgDsc2WD8F2q$o/vzA4myCqZZ36bUGsDY//8mKUYNZZaR0t4MFFSs+iM"
    email: user@example.com
    groups:
      - admins
      - dev
  # harry:
  #   disabled: false
  #   displayname: "Harry Potter"
  #   password: "$argon2id$v=19$m=65536,t=3,p=2$BpLnfgDsc2WD8F2q$o/vzA4myCqZZ36bUGsDY//8mKUYNZZaR0t4MFFSs+iM"
  #   email: harry.potter@authelia.com
  #   groups: []
  # bob:
  #   disabled: false
  #   displayname: "Bob Dylan"
  #   password: "$argon2id$v=19$m=65536,t=3,p=2$BpLnfgDsc2WD8F2q$o/vzA4myCqZZ36bUGsDY//8mKUYNZZaR0t4MFFSs+iM"
  #   email: bob.dylan@authelia.com
  #   groups:
  #     - dev
  # james:
  #   disabled: false
  #   displayname: "James Dean"
  #   password: "$argon2id$v=19$m=65536,t=3,p=2$BpLnfgDsc2WD8F2q$o/vzA4myCqZZ36bUGsDY//8mKUYNZZaR0t4MFFSs+iM"
  #   email: james.dean@authelia.com
```
</details>

---

### **Installation**

- [Official Documentation](https://www.authelia.com/configuration/prologue/introduction/) | Authelia.com

!!! Info "This guide uses Docker with docker compose"

First navigate to `/opt/docker-appdata/compose-files/authelia`

Create the config folder

```shell
mkdir config
```

Change directory into the newly made config folder

```shell
cd config
```

Create the `configuration.yml` file

```shell
nano configuration.yml
```

Enter the content from the [configuration.yml](https://docs.rognheim.no/tools-and-applications/compose-files/authelia/#configurationyml) template and make the necessary edits

Create the user_database.yml file

```shell
nano user_database.yml
```

Enter the content from the [user_database.yml](https://docs.rognheim.no/tools-and-applications/compose-files/authelia/#user_databaseyml) template and make the necessary edits

Hash your password with the built-in hashing tool provided by the authelia docker container

```shell
docker run authelia/authelia:latest authelia crypto hash generate argon2 --password 'your_password'
```

Copy and paste the output to the assigned user in user_database.yml

Create the secrets folder

```shell
mkdir secrets
```

Change directory into the newly made secrets folder

```shell
cd secrets
```

Now we need to create the four secret files that we have declared in the environment of the docker-compose file for authelia

It is strongly recommended the passwords for jwt_secrets, encryption_key, and session_secret is a random alphanumeric string with 64 or more characters. This can also be generated with the built-in password tool provided by the authelia docker container.

```shell
docker run authelia/authelia:latest authelia crypto rand --length 64 --charset alphanumeric
```

First create the jwt_secrets file and enter your random alphanumeric string with 64 or more characters 

```shell
nano jwt_secrets
```

Then create the encryption_key file and enter your random alphanumeric string with 64 or more characters

```shell
nano encryption_key
```

Then create the session_secret file and enter your random alphanumeric string with 64 or more characters

```shell
nano session_secret
```

And lastly create the smtp_password file and enter your app password from google or another smtp provider

```shell
nano smtp_password
```

It is strongly recommended that you ensure the permissions of the secret files are appropriately set so that other users or processes cannot access this file. Generally the UNIX permissions that are appropriate are 0600.

```shell
chmod 600 jwt_secrets && chmod 600 encryption_key && chmod 600 session_secret && chmod 600 smtp_password
```

Navigate back to /opt/docker-appdata/compose-files/authelia

Create the docker-compose.yml file

```shell
nano docker-compose.yml
```

Enter the content from the [docker-compose.yml](https://docs.rognheim.no/tools-and-applications/compose-files/authelia/#docker-composeyml) template and make the necessary edits

Spin up the container

```shell
docker compose up -d
```

---

## **Enable authelia on other containers**

To enable authelia on any other container just add the following label to the docker-compose.yml file of that container

```yaml
    labels:
      - traefik.http.routers.traefik.middlewares=authelia@docker # Needed to enable authelia on this domain
```

---

## **Forwarded Headers**

- [Authelia Forwarded Headers Documentation](https://www.authelia.com/integration/proxies/fowarded-headers/)
- [Traefik Forwarded Headers Documentation](https://doc.traefik.io/traefik/routing/entrypoints/#forwarded-headers)

To ensure the proper handling of the X-Forwarded-For header and to prevent IP address forgery, you should use Cloudflare's "True-Client-IP" header. This header is added by Cloudflare to all requests, and contains the actual client IP address.

**1.** Log in to your Cloudflare account
**2.** Select the domain for which you want to configure the settings
**3.** Click on the "Network" tab
**4.** Scroll down to the "IP Geolocation" section and make sure it's set to "On"

Update your Traefik configuration to use the "X-Real-Ip" headers found in the cloudflare documentation: [Cloudflare IP Ranges](https://www.cloudflare.com/ips/)

```yaml
entryPoints:
  http:
    address: :80
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
          permanent: true
  https:
    address: :443
    forwardedHeaders:
      trustedIPs:      
        - "192.168.0.0/20" # Local IP Ranges
        - "103.21.244.0/22" # Cloudflare IPv4 Ranges
        - "103.22.200.0/22" 
        - "103.31.4.0/22"
        - "104.16.0.0/13"
        - "104.24.0.0/14"
        - "108.162.192.0/18"
        - "131.0.72.0/22"
        - "141.101.64.0/18"
        - "162.158.0.0/15"
        - "172.64.0.0/13"
        - "173.245.48.0/20"
        - "188.114.96.0/20"
        - "190.93.240.0/20"
        - "197.234.240.0/22"
        - "198.41.128.0/17"
        - "2400:cb00::/32" # Cloudflare IPv6 Ranges
        - "2606:4700::/32"
        - "2803:f800::/32"
        - "2405:b500::/32"
        - "2405:8100::/32"
        - "2a06:98c0::/29"
        - "2c0f:f248::/32"
      insecure: false
```

Recreate the traefik container and now clients visiting your domain will have their real IP address visible in the logs

!!! Warning "It is essential to make sure that this works as intended before spinning up the crowdsec container, or else you wont have any protection againts malicious actors."

---

### **Known issues**

### Authelia not working correctly

- [x] Solved by making sure that the authelia container is fully up and running before starting the traefik container. If you have not disabled healthchecks the authelia container can take a few moments to boot before it is fully up and running. You could use the "depends_on" variable, but then you have to have traefik and authelia in the same docker-compose.yml stack.

<!-- ---

#### Configure updates



---

### **Setting up alerts** -->---

### **Post-install tasks**

- To be documented.

