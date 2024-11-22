# Bluesky PDS on Fly.io

This guide helps you set up a Personal Data Server ([PDS](https://github.com/bluesky-social/pds)) for Bluesky on Fly.io.

## Prerequisites

- A Fly.io account
- Fly CLI installed
- A domain name (e.g., my.atproto.app)
- An S3-compatible storage service (e.g., Cloudflare R2 or Tigris)

## Installation Steps

### 1. Install Fly CLI

On macOS

```bash
brew install flyctl
```

On Linux
```bash
curl -L https://fly.io/install.sh | sh
```

### 2. Login to Fly.io

```bash
fly auth login
```

### 3. Adapt the fly.toml

[fly.toml](fly.toml) is the configuration file for the PDS. Adapt it to your needs.

### 4. Set Required Secrets

```bash
fly secrets set \
PDS_JWT_SECRET=$(openssl rand -hex 16) \
PDS_ADMIN_PASSWORD=$(openssl rand -hex 16) \
PDS_PLC_ROTATION_KEY_K256_PRIVATE_KEY_HEX=$(openssl ecparam -name secp256k1 -genkey -noout -outform DER | tail -c+8 | head -c32 | xxd -p -c32)
```

### 5. Set S3 credentials (for Cloudflare R2)

```bash
fly secrets set \
PDS_BLOBSTORE_S3_ACCESS_KEY_ID='your-r2-access-key-id' \
PDS_BLOBSTORE_S3_SECRET_ACCESS_KEY='your-r2-secret-access-key'
```

Alternatively, you can use [Tigris](https://fly.io/docs/tigris/) for blob storage.

### 6. Email

Set SMTP configuration (using [Resend](https://resend.com/) as example)

```bash
fly secrets set \
PDS_EMAIL_SMTP_URL="smtps://resend:re_YOUR_API_KEY@smtp.resend.com:465/" \
PDS_EMAIL_FROM_ADDRESS="noreply@your-domain.com"
```

### 7. Create a mount

Create a [mount](https://fly.io/docs/reference/configuration/#the-mounts-section) to persist data, needs at least 2 nodes.

```bash
fly mounts create pds_data -r <your-region> -n 2
```

### 8. Deploy

```bash
fly deploy
```

### 9. Configure DNS

Add your domain to Fly.io

```bash
fly certs add your-domain.com
```

Get IP addresses to configure at your DNS provider

```bash
fly ips list
```

Add these records at your DNS provider:

```
A Record: your-domain.com -> points to Fly's IPv4 address
AAAA Record: your-domain.com -> points to Fly's IPv6 address
```

### 10. Request Network Crawl

request a network crawl:

```bash
curl \
--fail \
--silent \
--show-error \
--request POST \
--header "Content-Type: application/json" \
--data '{"hostname": "your-domain.com"}' \
"https://bsky.network/xrpc/com.atproto.sync.requestCrawl"
```


### 11. Create First Account

```bash
fly ssh console -a my-pds-app
```

Get the admin password

```bash
env | grep PDS_ADMIN_PASSWORD
```

Note the password, you will need it to create your account. Exit the console.

Create an invite code with curl

```bash
curl \
--fail \
--silent \
--show-error \
--user "admin:<your-password>" \
--request POST \
--header "Content-Type: application/json" \
--data '{"useCount": 1}' \
"https://<your-domain>/xrpc/com.atproto.admin.createInviteCode"
```

Note the invite code, you will need it to create your account.

Create an account with curl

```bash
curl \
--fail \
--silent \
--show-error \
--request POST \
--header "Content-Type: application/json" \
--data '{"email": "<your-email>", "handle": "<your-handle>", "password": "<your-password>", "inviteCode": "<your-invite-code>"}' \
"https://<your-domain>/xrpc/com.atproto.server.createAccount"
```
