# LuxeStore — Three-Tier Web Application

<p align="center">
  <img src="https://img.shields.io/badge/Architecture-Three--Tier-6C63FF?style=for-the-badge" alt="Three-Tier Architecture" />
  <img src="https://img.shields.io/badge/Frontend-Nginx%20%2B%20HTML5%2FCSS%2FJS-009639?style=for-the-badge&logo=nginx&logoColor=white" alt="Frontend Nginx" />
  <img src="https://img.shields.io/badge/Backend-Node.js%2022%20%2B%20Express-339933?style=for-the-badge&logo=node.js&logoColor=white" alt="Node.js 22" />
  <img src="https://img.shields.io/badge/Database-MySQL%208.0+-4479A1?style=for-the-badge&logo=mysql&logoColor=white" alt="MySQL 8.0+" />
  <img src="https://img.shields.io/badge/Process%20Manager-PM2-2B037A?style=for-the-badge&logo=pm2&logoColor=white" alt="PM2" />
</p>

**LuxeStore** is a modern, responsive, full-stack e-commerce web application engineered with a **Decoupled Three-Tier Architecture**. Each tier (Presentation, Application, and Data) is isolated, allowing the system to be deployed either on a **single Linux VM** for rapid testing or distributed across **three separate VMs** in a production cloud environment.

---

## Architecture Overview

```text
+-----------------------------------------------------------------------------+
|                       TIER 1: PRESENTATION (FRONTEND)                       |
|   • Nginx Web Server (Port 80)                                              |
|   • Static SPA-style Frontend (HTML5, CSS3 Light/Dark Themes, Vanilla JS)   |
|   • Storefront & Admin Portal (/admin/)                                     |
|   • Reverse Proxy: Forwards /api/* requests to Application Tier             |
+--------------------------------------+--------------------------------------+
                                       | HTTP REST (:80/api/ -> :3000)
+--------------------------------------v--------------------------------------+
|                    TIER 2: APPLICATION / LOGIC (BACKEND)                    |
|   • Node.js 22.x & Express.js REST API (Port 3000)                          |
|   • Managed by PM2 Process Manager (Auto-restart, zero downtime)            |
|   • Stateless JWT Authentication & Role-Based Access (Customer / Admin)     |
|   • Inventory Validation & ACID Transactional Order Processing              |
|   • Security: Helmet headers, CORS policies, Express Rate Limiter           |
+--------------------------------------+--------------------------------------+
                                       | MySQL Protocol (Port 3306)
+--------------------------------------v--------------------------------------+
|                           TIER 3: DATA (DATABASE)                           |
|   • MySQL 8.0+ / MariaDB (Port 3306, utf8mb4)                               |
|   • Relational Schema: users, categories, products, cart_items, orders      |
|   • Foreign Key Constraints, Cascading Rules, Full-Text Search Indexes      |
|   • Dedicated Application User: ecommerce_user                              |
+-----------------------------------------------------------------------------+
```

### Deployment Modes

- **Single-VM Mode**: All three tiers run on one Ubuntu machine. Inter-service communication happens over `localhost` (`127.0.0.1`). Ideal for quick local testing and evaluation.
- **Multi-VM Mode**: Each tier runs on its own isolated VM (Frontend VM, Backend VM, Database VM). Communication happens across private network IPs.

---

## Directory Structure

```text
3-tier-app/
├── frontend/                    # Tier 1: Presentation
│   ├── admin/                   # Admin Portal (Dashboard, Products, Orders, Categories)
│   ├── css/                     # Design system & dark/light mode stylesheets
│   ├── js/                      # Modular client logic, API client, & cart manager
│   ├── *.html                   # Storefront pages (index, product, cart, checkout, orders)
│   └── nginx.conf               # Nginx reverse proxy & static asset configuration
├── backend/                     # Tier 2: Application / API
│   ├── config/                  # Database connection pool (mysql2/promise)
│   ├── middleware/              # Auth guards (JWT), Admin checks, Error handlers
│   ├── routes/                  # Modular REST endpoints (auth, products, cart, orders, admin)
│   ├── utils/                   # Input validators & sanitizers
│   ├── .env.example             # Backend environment variable template
│   ├── server.js                # Express application entrypoint
│   └── package.json             # Node.js dependencies
└── db/                          # Tier 3: Database
    ├── init.sql                 # Database, dedicated user, & table initialization
    └── seed.sql                 # Sample catalog items, categories, & demo admin
```

---

## Setup & Deployment Guide

> **Note**: For **Single-VM setup**, execute all steps on the same machine. For **Multi-VM setup**, run each tier on its designated VM and replace IP placeholders accordingly.

---

### Tier 3: Database Setup (MySQL)

On the Database VM (or your single VM):

```bash
# 1. Update system & install MySQL Server
sudo apt update && sudo apt upgrade -y
sudo apt install -y mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql

# 2. Run security setup
sudo mysql_secure_installation

# 3. Configure Database & Application User
# Edit db/init.sql to set your own password for 'ecommerce_user'
sudo mysql -u root < db/init.sql

# 4. Seed Categories, Products, and Demo Admin
sudo mysql -u root < db/seed.sql
```

> **For Multi-VM Deployment Only**:
> Enable MySQL to accept connections from the Backend VM:
> 1. Edit `/etc/mysql/mysql.conf.d/mysqld.cnf`:
>    ```ini
>    bind-address = 0.0.0.0
>    ```
> 2. Restart MySQL: `sudo systemctl restart mysql`
> 3. Ensure your Cloud Firewall / NSG allows inbound **Port 3306** strictly from the **Backend VM's Private IP**.

---

### Tier 2: Backend Setup (Node.js + PM2)

On the Backend VM (or your single VM):

```bash
# 1. Install Node.js 22.x LTS & PM2
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2

# 2. Clone repository & install dependencies
cd ~
git clone https://github.com/<your-username>/3-tier-app.git
cd ~/3-tier-app/backend
npm install

# 3. Configure Environment Variables
cp .env.example .env
nano .env
```

**Configure `.env`**:

```env
PORT=3000
NODE_ENV=production

# Database Host:
# - Single-VM: Use 127.0.0.1
# - Multi-VM:  Use <DATABASE_VM_PRIVATE_IP>
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=ecommerce_user
DB_PASSWORD=YourStrongPasswordHere!
DB_NAME=ecommerce_store

JWT_SECRET=your_super_secret_jwt_key_here
JWT_EXPIRES_IN=7d

# CORS:
# - Single-VM: Use *
# - Multi-VM:  Use http://<FRONTEND_PUBLIC_IP_OR_DOMAIN>
CORS_ORIGIN=*
```

**Start the API with PM2**:

```bash
pm2 start server.js --name ecommerce-api
pm2 save
pm2 startup
```

*(Copy and execute the `sudo env PATH=...` command printed by `pm2 startup` to enable automatic restart on server reboot).*

> **For Multi-VM Deployment Only**:
> Ensure your Cloud Firewall / NSG allows inbound **Port 3000** strictly from the **Frontend VM's Private IP**.

---

### Tier 1: Frontend Setup (Nginx)

On the Frontend VM (or your single VM):

```bash
# 1. Install & enable Nginx
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx

# 2. Deploy frontend static assets
sudo mkdir -p /var/www/html/frontend
sudo cp -r ~/3-tier-app/frontend/* /var/www/html/frontend/
sudo chown -R www-data:www-data /var/www/html/frontend/

# 3. Configure Nginx Reverse Proxy
sudo cp ~/3-tier-app/frontend/nginx.conf /etc/nginx/sites-available/default
```

> **For Multi-VM Deployment Only**:
> Edit `/etc/nginx/sites-available/default` so `/api/` proxies to the Backend VM's private IP:
> ```nginx
> location /api/ {
>     proxy_pass http://<BACKEND_VM_PRIVATE_IP>:3000;
>     ...
> }
> ```
> *(For Single-VM setup, leave `proxy_pass http://localhost:3000;` as is).*

**Test and reload Nginx**:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Firewall & Network Port Map (Multi-VM)

| Tier | Port | Protocol | Source / Inbound Rule | Purpose |
|---|---|---|---|---|
| **Frontend VM** | `80` (HTTP) | TCP | `0.0.0.0/0` (Internet) | Public web storefront & Nginx |
| **Frontend VM** | `443` (HTTPS)| TCP | `0.0.0.0/0` (Internet) | Optional SSL/TLS with Let's Encrypt |
| **Backend VM** | `3000` | TCP | `Frontend_VM_Private_IP` | Express REST API |
| **Database VM** | `3306` | TCP | `Backend_VM_Private_IP` | MySQL Database connectivity |

---

## Testing & Verifying the Application

1. Open your browser and navigate to:
   ```text
   http://<FRONTEND_VM_PUBLIC_IP>/
   ```
2. **Customer Flow**:
   - Browse products and filter by category.
   - Click **"Register"** to create a customer account.
   - Add items to the cart and complete checkout.
   - View your placed order in **"My Orders"**.
3. **Admin Portal**:
   - Navigate to `http://<FRONTEND_VM_PUBLIC_IP>/admin/dashboard.html`
   - Log in with your admin credentials to view KPI metrics, catalog management, and order status updates.
