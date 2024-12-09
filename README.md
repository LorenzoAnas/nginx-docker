# Nginx Reverse Proxy with Docker

This guide will help you set up an Nginx reverse proxy using Docker. A reverse proxy is a server that sits between client devices and backend servers, forwarding client requests to the appropriate backend server. This setup is useful for load balancing, securing traffic with SSL, and managing multiple services on a single server.

## Prerequisites

- Docker and Docker Compose installed on your server.
- A domain name (e.g., `your-domain.com`).
- Access to your domain's DNS settings (e.g., Namecheap).

## Setup

### Step 1: Clone the Repository

```sh
git clone https://github.com/LorenzoAnas/nginx-docker.git
cd nginx-docker
```

### Step 2: Configure Nginx

Edit the `nginx/conf.d/default.conf` file to match your setup. Here is an example configuration:

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        proxy_pass http://your-backend-server:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate     /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    location / {
        proxy_pass http://your-backend-server:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Step 3: Obtain SSL Certificates

We will use Certbot to obtain SSL certificates. Update the `docker-compose.yml` file with your email and domain:

```yaml
certbot:
  image: certbot/certbot:latest
  container_name: certbot
  volumes:
    - ./nginx/certs:/etc/letsencrypt
  command: certonly --manual --preferred-challenges dns --agree-tos --email your-email@example.com --no-eff-email -d your-domain.com
```

### Step 4: Add TXT Record for DNS Challenge

To verify your domain ownership, you need to add a TXT record to your DNS settings. For example, on Namecheap:

1. Log in to your Namecheap account.
2. Go to "Domain List" and click "Manage" next to your domain.
3. Navigate to the "Advanced DNS" tab.
4. Add a new "TXT Record" with the name `_acme-challenge.your-domain.com` and the value provided by Certbot.

### Step 5: Start the Containers

Run the following command to start the Nginx and Certbot containers:

```sh
docker-compose up -d
```

### Step 6: Verify SSL Setup

After Certbot successfully obtains the certificates, you can verify the SSL setup by visiting `https://your-domain.com`.

### Step 7: Automate Certificate Renewal with Crontab

To ensure your SSL certificates are renewed automatically before they expire, you can set up a cron job.

1. Open your crontab:
   ```sh
   crontab -e
   ```

2. Add the following line to schedule the renewal process daily:
   ```sh
   0 3 * * * cd /path/to/nginx-docker && docker-compose run certbot renew >> /path/to/renewal.log 2>&1 && docker-compose restart nginx
   ```

   - Replace `/path/to/nginx-docker` with the directory where your `docker-compose.yml` file is located.
   - Logs from the renewal process will be saved to `/path/to/renewal.log`.

3. Save and exit the crontab file.

The cron job will run daily at 3:00 AM, checking if renewal is required and restarting the Nginx service to apply updated certificates.

## Tips

- Ensure your backend server is running and accessible from the Nginx container.
- Regularly renew your SSL certificates using Certbot.

## Why Proxying is Important

Proxying is essential for several reasons:

- **Load Balancing**: Distribute incoming traffic across multiple backend servers to ensure no single server is overwhelmed.
- **Security**: Terminate SSL/TLS connections at the proxy server to secure traffic between clients and the proxy.
- **Centralized Management**: Manage multiple services and domains from a single point, simplifying configuration and maintenance.

By setting up an Nginx reverse proxy with Docker, you can efficiently manage and secure your web services.
