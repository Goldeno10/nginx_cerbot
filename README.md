# How to Set Up SSL with Let's Encrypt, Nginx, and Docker on a Linux VPS

In this guide, we’ll set up **SSL** using **Let's Encrypt** with **Nginx** and **Docker** on a Linux VPS. We’ll go through each step, from creating a temporary Nginx setup to avoid errors, to configuring SSL and automating certificate renewal. This example uses the domain `datacube.uxlivinglab.online`.

---

## Prerequisites

1. **Linux VPS** with root access.
2. **Docker** and **Docker Compose** installed.
3. A **domain name** pointing to your VPS’s IP address.
4. **Nginx** and **Certbot** to handle SSL and reverse proxy.

---

## Step 1: Set Up the Directory Structure

Create a directory to house your Docker and configuration files.

```bash
mkdir ~/nginx-certbot
cd ~/nginx-certbot
```

## Step 2: Create a Docker Compose File for Nginx and Certbot

In this step, we’ll set up **Docker Compose** to manage both Nginx and Certbot. Create a `docker-compose.yml` file in your project directory with the following content:

```yaml
version: "3"
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: unless-stopped
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    ports:
      - "80:80"
      - "443:443"
    networks:
      - app-network

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do sleep 6h & wait $${!}; certbot renew; done'"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### This configuration file does the following:

- **Nginx** is set up to listen on both HTTP (port 80) and HTTPS (port 443).
- **Certbot** handles SSL certificate generation and renewals every 6 hours.
- The **volumes** allow Nginx and Certbot to share SSL certificates and challenge files.
- Both services are connected to a custom network, `app-network`, which will allow them to communicate securely.

## Step 3: Temporary Nginx Configuration to Avoid Errors

Since the SSL certificate is not available yet, we’ll set up a temporary Nginx configuration to avoid errors. This configuration will serve a simple static page on HTTP until SSL is ready.

1. Create a directory for the Nginx configuration:

   ```bash
   mkdir -p nginx/conf.d
   ```

2. Create a temporary configuration file named `nginx/conf.d/django.conf` with the following content:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Serve a basic HTML page for now
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

This temporary configuration tells Nginx to:

- Listen on port 80 (HTTP) and handle Let’s Encrypt challenges.
- Serve a static HTML page until SSL is set up, which prevents errors from missing SSL files.

## Step 4: Create a Basic HTML Page

For the temporary setup, create a simple HTML file that Nginx can serve while SSL is being configured. This will ensure Nginx runs without errors.

Run the following command to create the HTML file:

```bash
echo "<html><body><h1>Nginx is running, SSL setup in progress.</h1></body></html>" | sudo tee /usr/share/nginx/html/index.html
```

This creates an HTML file at `/usr/share/nginx/html/index.html`, which Nginx will serve as a placeholder until SSL is fully configured.

## Step 5: Bring Up the Docker Containers

With the configurations in place, start the Nginx and Certbot containers using Docker Compose.

Run the following command to bring up the containers in detached mode:

```bash
docker-compose up -d
```

To verify that both Nginx and Certbot are running, list the active containers:

```bash
docker ps
```

You should see output showing both containers up and running, similar to this:

```bash
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS                                       NAMES
abc123456789   nginx:latest      "/docker-entrypoint.…"   1 minute ago     Up 1 minute     0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp    nginx
def123456789   certbot/certbot   "/bin/sh -c 'trap ex…"   1 minute ago     Up 1 minute                                                 certbot
```

This indicates that Nginx is now serving content over HTTP and Certbot is ready to handle SSL certificate requests.

## Step 6: Obtain SSL Certificates from Let's Encrypt

With Nginx running and serving content over HTTP, we can now request SSL certificates from Let’s Encrypt using Certbot.

Run the following command to obtain an SSL certificate:

```bash
docker run -it --rm \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  certbot/certbot certonly \
  --webroot --webroot-path=/var/www/certbot \
  --email your-email@example.com \
  --agree-tos \
  --no-eff-email \
  -d yourdomain.com \
  -d www.yourdomain.com
```

`Note: Replace your-email@example.com with your email and yourdomain.com with your actual domain.`

If successful, you should see output similar to this:

```bash
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/yourdomain.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/yourdomain.com/privkey.pem
```

This indicates that Certbot has successfully generated the SSL certificate, which is now stored in the `/etc/letsencrypt/live/yourdomain.com/` directory.

## Step 7: Final Nginx Configuration with SSL

Now that the SSL certificates are ready, update the Nginx configuration to enable SSL and redirect HTTP traffic to HTTPS.

Edit the `nginx/conf.d/django.conf` file and replace the content with the following:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Redirect all HTTP traffic to HTTPS
    return 301 https://$host$request_uri;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}

server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://django:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

`Note: Replace yourdomain.com with your actual domain.`

### This configuration:

- Redirects all HTTP traffic to HTTPS.
- Configures Nginx to use the SSL certificate and key files generated by Let’s Encrypt.
- Proxies traffic to the Django app on port 8000 (replace this with your application’s actual port if needed).
  With this setup, your site is ready to serve content securely over HTTPS.

## Step 8: Reload Nginx with SSL Configuration

Now that the final Nginx configuration is set up with SSL, reload Nginx to apply the changes:

```bash
docker-compose exec nginx nginx -s reload
```

This command tells Nginx to reload its configuration files without needing to restart the container. Your site should now be live and serving content over HTTPS.

To confirm, navigate to:

```
https://yourdomain.com
```

If everything is configured correctly, your site will now load securely with the SSL certificate.

## Step 9: Test SSL Setup

With SSL configured and Nginx reloaded, it’s time to test that your site is accessible over HTTPS.

1. Open a browser and go to:

```
https://yourdomain.com
```

2. Verify that your site loads with a secure connection. Look for the padlock icon in the browser’s address bar, indicating that the SSL certificate is active.

3. (Optional) To further verify your SSL setup, you can use an SSL checker like [SSL Labs](https://www.ssllabs.com/ssltest/) by entering your domain name. This tool will provide details on the SSL configuration, including certificate validity and potential issues.

If everything is set up correctly, your site is now secure and accessible over HTTPS.

2. Verify that your site loads with a secure connection. Look for the padlock icon in the browser’s address bar, indicating that the SSL certificate is active.

3. (Optional) To further verify your SSL setup, you can use an SSL checker like [SSL Labs](https://www.ssllabs.com/ssltest/) by entering your domain name. This tool will provide details on the SSL configuration, including certificate validity and potential issues.

If everything is set up correctly, your site is now secure and accessible over HTTPS.

## Step 10: Set Up Automatic SSL Renewal

To prevent your SSL certificate from expiring, set up automatic renewal. Run this command to manually test renewal:

```bash
docker-compose exec certbot certbot renew --quiet
```




+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# How to Renew Let's Encrypt SSL Certificate for Nginx in Docker

## Overview
If your Let's Encrypt SSL certificate has expired and is not renewing automatically in your **Dockerized Nginx setup**, this guide will walk you through manually renewing the certificate and setting up automatic renewal.

---

## Prerequisites
Before proceeding, ensure you have:
- A **Linux VPS** with root access.
- **Docker** and **Docker Compose** installed.
- A running **Nginx container** with Let's Encrypt SSL setup.
- A valid **domain name** pointing to your VPS.

---

## Step 1: Check the Expired Certificate
To check if your SSL certificate has expired, run:

```bash
sudo openssl x509 -noout -dates -in certbot/conf/live/YOUR_DOMAIN/fullchain.pem
```

If the expiration date has passed, continue with the renewal steps.

---

## Step 2: Check Let's Encrypt Logs
To diagnose why the certificate did not renew automatically, check the logs:

```bash
docker logs certbot
```

Look for any errors related to **challenge validation failures** or rate limits.

---

## Step 3: Verify Nginx Configuration for Let's Encrypt
Your **nginx/conf.d/django.conf** should contain this configuration to allow Let's Encrypt challenges before redirecting traffic to HTTPS:

```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        allow all;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name YOUR_DOMAIN;

    ssl_certificate /etc/letsencrypt/live/YOUR_DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/YOUR_DOMAIN/privkey.pem;

    location / {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Restart Nginx to apply changes:
```bash
docker-compose restart nginx
```

---

## Step 4: Test If Let's Encrypt Can Access Challenge Files
Create a test challenge file:

```bash
mkdir -p certbot/www/.well-known/acme-challenge
echo "test123" > certbot/www/.well-known/acme-challenge/test-file
```

Try accessing:
```
http://YOUR_DOMAIN/.well-known/acme-challenge/test-file
```

- If you see **test123**, Certbot should work.
- If you get a **404 error**, ensure Nginx is properly configured.

---

## Step 5: Manually Renew the SSL Certificate
If your certificate has expired, run:

```bash
docker run --rm \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  certbot/certbot renew
```

If successful, restart Nginx to load the new certificate:

```bash
docker-compose restart nginx
```

---

## Step 6: Verify SSL Renewal
Check if the certificate has been updated:

```bash
sudo openssl x509 -noout -dates -in certbot/conf/live/YOUR_DOMAIN/fullchain.pem
```

---

## Step 7: Automate SSL Renewal
To ensure certificates renew automatically, create a **cron job**:

```bash
crontab -e
```

Add this line to renew every Sunday at midnight:
```bash
0 0 * * 7 docker run --rm -v /etc/letsencrypt:/etc/letsencrypt certbot/certbot renew && docker-compose restart nginx
```

Save and exit.

---

## Step 8: Final Checks
- Visit **https://YOUR_DOMAIN** to confirm SSL is working.
- Check Certbot logs for any future issues:
  ```bash
  cat /var/log/letsencrypt/letsencrypt.log
  ```
- Test auto-renewal manually:
  ```bash
  docker run --rm -v "$(pwd)/certbot/conf:/etc/letsencrypt" -v "$(pwd)/certbot/www:/var/www/certbot" certbot/certbot renew --dry-run
  ```

---

## 🎉 Done! 
Your SSL certificate is now renewed and will auto-renew in the future. 🚀



