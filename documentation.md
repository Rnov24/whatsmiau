# WhatsMiau Documentation

WhatsMiau is a high-performance, lightweight WhatsApp backend service built with Go. It wraps the `whatsmeow` library and exposes a REST API compatible with the **Evolution API**, making it a drop-in replacement with extremely low memory overhead.

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Local Setup Guide](#2-local-setup-guide)
3. [Railway Deployment Guide](#3-railway-deployment-guide)
4. [Connecting a WhatsApp Number](#4-connecting-a-whatsapp-number)
5. [API Reference Quickstart](#5-api-reference-quickstart)
6. [Troubleshooting & Proxy Routing](#6-troubleshooting--proxy-routing)

---

## 1. Prerequisites

To run this project locally, ensure you have the following installed:
- **Go** (version 1.24 or higher)
- **Redis** (running locally on port `6379`)
- **FFmpeg** (installed and in your system PATH, required for audio transcoding)

---

## 2. Local Setup Guide

1. **Clone and Navigate:**
   ```bash
   git clone https://github.com/Rnov24/whatsmiau.git
   cd whatsmiau
   ```

2. **Initialize Environment Variables:**
   Copy the example file to `.env`:
   ```bash
   cp .env.example .env
   ```

3. **Install Dependencies:**
   ```bash
   go mod tidy
   ```

4. **Run Unit Tests:**
   ```bash
   go test ./...
   ```

5. **Start the Application:**
   ```bash
   go run main.go
   ```
   * The server will boot up and bind to the port defined in your `.env` (default is `8081`).
   * Access Swagger Documentation: `http://localhost:8081/swagger/index.html`
   * Access Manager Dashboard: `http://localhost:8081/manager/`

---

## 3. Railway Deployment Guide

Because WhatsApp blocks server IP ranges from cloud hosting providers like Railway, you must route your connection traffic through a proxy server (HTTP/SOCKS5).

### Step 1: Create a Railway Project
1. Link your GitHub repository to Railway and select deploy.
2. Add a custom **Redis** service and a custom **PostgreSQL** service to your canvas.

### Step 2: Configure Environment Variables
Copy and paste this variables configuration into the **Raw Editor** of your `whatsmiau` service variables tab:

```ini
API_KEY=b79670f7ee35089549cac90a4315f3edd3993652084a7416e7c30a16d000dcfc
DB_URL=postgres://postgres:iLXKCbDwWQdQUyoUNeBzorzNaAVbExBm@postgres.railway.internal:5432/railway?sslmode=disable
DIALECT_DB=postgres
PORT=8080
REDIS_PASSWORD=osjmvZLAQghKjvPDWfOHckNOmlarnrJC
REDIS_TLS=false
REDIS_URL=redis.railway.internal:6379
PROXY_ADDRESSES=HTTP://username:password@proxyhost:proxyport
MANAGER_URL=https://your-public-railway-domain.up.railway.app
```
*Note: Replace `PROXY_ADDRESSES` with your own proxy connection string, and `MANAGER_URL` with your generated Railway domain.*

---

## 4. Connecting a WhatsApp Number

### Method A: Web Manager Dashboard (Recommended)
1. Navigate to `https://your-public-railway-domain.up.railway.app/manager/`.
2. Input your `API_KEY` to log in.
3. Click **+ Nova Instância** (+ New Instance) -> Input an instance name (e.g. `main`) -> Click **Criar** (Create).
4. Click on your created instance -> Click **Conectar** (Connect).
5. Open WhatsApp on your phone -> **Linked Devices** -> **Link a Device** -> Scan the QR Code.

### Method B: API (Pairing Code Option)
You can retrieve an 8-digit pairing code to link your phone number instead of scanning a QR code:
```bash
curl -X GET 'https://your-public-railway-domain.up.railway.app/v1/instance/connect/main?number=6281573185961' \
  -H 'apikey: YOUR_API_KEY'
```
Enter the returned pairing code in your phone's WhatsApp application (**Linked Devices** -> **Link with phone number instead**).

---

## 5. API Reference Quickstart

### Create Instance
* **POST** `/v1/instance/create`
* **Headers:** `apikey: <api_key>`
* **Body:**
  ```json
  {
    "instanceName": "main"
  }
  ```

### Get Connection QR Code
* **GET** `/v1/instance/connect/{instanceName}`
* **Headers:** `apikey: <api_key>`
* **Response:** Base64-encoded QR code png image data.

### Send Text Message
* **POST** `/v1/instance/{instanceName}/message/text`
* **Headers:** `apikey: <api_key>`
* **Body:**
  ```json
  {
    "number": "6281573185961",
    "text": "Hello world!"
  }
  ```

### Update Webhook Configuration
* **PUT** `/v1/instance/update/{instanceName}`
* **Headers:** `apikey: <api_key>`
* **Body:**
  ```json
  {
    "webhook": {
      "enabled": true,
      "url": "https://your-api-domain.com/webhook",
      "base64": false,
      "events": [
        "MESSAGES_UPSERT",
        "CONNECTION_UPDATE"
      ]
    }
  }
  ```

---

## 6. Troubleshooting & Proxy Routing

### 1. `failed to start sqlstore` or `failed to connect to redis`
* Ensure your `DB_URL` and `REDIS_URL` point to the correct internal domains: `postgres.railway.internal:5432` and `redis.railway.internal:6379`.
* Check the **Variables** tab of your Postgres/Redis containers to ensure the passwords (`POSTGRES_PASSWORD` and `REDISPASSWORD`) match what you configured in the `whatsmiau` variables.

### 2. Connection Timeout / `context deadline exceeded` during QR Generation
* This means WhatsApp is blocking Railway's IP ranges.
* Make sure `PROXY_ADDRESSES` is set correctly and the scheme is in lowercase (`http://...` or `socks5://...`). 
* Note: If you create an instance *before* setting the `PROXY_ADDRESSES` variable, you must delete and recreate the instance (or update its settings) so the database records the proxy details.
