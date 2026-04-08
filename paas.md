# Studi Kasus & Tutorial Lengkap: Membangun Mini Platform PaaS dengan Docker SDK

Platform as a Service (PaaS) adalah model komputasi awan yang memungkinkan developer untuk *deploy* dan menjalankan aplikasi tanpa harus mengelola infrastruktur server di bawahnya. Proyek akhir ini mengajak Anda membangun **mini-PaaS sederhana namun fungsional** menggunakan Docker SDK sebagai fondasi utama. Platform ini akan mampu menerima *push* aplikasi dari Git, membangunnya menjadi *container image*, lalu menjalankannya di server yang sama, mirip dengan konsep **Heroku mini** yang populer.

> **Referensi Inspirasi**: Dokku adalah proyek *open-source* yang digambarkan oleh penciptanya sebagai "Docker-powered mini-Heroku in around 100 lines of Bash". Fitur-fitur utama Dokku meliputi *deployment* melalui Git, pembangunan otomatis, dan pengelolaan aplikasi dalam *container* yang berdiri sendiri.

Proyek ini akan mengadopsi pendekatan serupa namun membangunnya dari awal dengan Docker SDK, memberi Anda pemahaman mendalam tentang cara kerja platform modern di balik layar.

---

## 📚 Daftar Isi

- [Pendahuluan & Studi Kasus](#pendahuluan--studi-kasus)
- [Arsitektur Sistem](#arsitektur-sistem)
- [Prasyarat & Persiapan Lingkungan](#prasyarat--persiapan-lingkungan)
- [Tutorial Step-by-Step](#tutorial-step-by-step)
  - [Bab 1: Setup Control Plane](#bab-1-setup-control-plane)
  - [Bab 2: Membangun REST API untuk Platform](#bab-2-membangun-rest-api-untuk-platform)
  - [Bab 3: Integrasi Docker SDK](#bab-3-integrasi-docker-sdk)
  - [Bab 4: HTTP Routing & Reverse Proxy Dinamis](#bab-4-http-routing--reverse-proxy-dinamis)
  - [Bab 5: Deployment Pipeline (Git Push + Webhook)](#bab-5-deployment-pipeline-git-push--webhook)
  - [Bab 6: Sistem Registry Image](#bab-6-sistem-registry-image)
  - [Bab 7: Monitoring & Logging](#bab-7-monitoring--logging)
- [Testing & Validasi](#testing--validasi)
- [Kesimpulan & Next Steps](#kesimpulan--next-steps)

---

## Pendahuluan & Studi Kasus

Bayangkan Anda adalah seorang *platform engineer* di sebuah startup teknologi yang sedang berkembang pesat. Tim developer merasa kesulitan dengan alur *deployment* yang rumit—mereka harus menyiapkan server, menginstal dependensi, mengatur *environment variable*, dan mengelola proses aplikasi secara manual untuk setiap layanan.

Untuk mengatasi masalah ini, Anda ditugaskan untuk membangun **platform internal sederhana** yang memungkinkan developer untuk melakukan:

- **Push kode aplikasi** melalui Git, dan platform secara otomatis akan:
  1. Membangun aplikasi menjadi Docker *image*
  2. Menjalankan *container* dari *image* tersebut
  3. Memberikan URL publik yang dapat diakses

- **Mengelola beberapa aplikasi** secara bersamaan di satu server
- **Melihat status** setiap aplikasi (*running*, *stopped*, *failed*)
- **Menghentikan/menghapus aplikasi** dengan mudah
- **Memonitor resource** yang digunakan oleh setiap aplikasi

Studi kasus ini akan menggunakan **Node.js + Express** untuk *control plane* dan **Docker SDK for Python** untuk berinteraksi dengan Docker daemon. Pendekatan ini memberikan keseimbangan antara kemudahan implementasi dan kemampuan untuk melakukan operasi *container* secara *programmatic*.

---

## Arsitektur Sistem

Berikut adalah arsitektur lengkap dari platform yang akan kita bangun:

```
┌─────────────────────────────────────────────────────────────────┐
│                         DEVELOPER WORKSTATION                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  git push origin main                                    │    │
│  └──────────────────────────┬──────────────────────────────┘    │
└─────────────────────────────┼────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MINI-PaaS PLATFORM                          │
│                                                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Git HTTP   │    │   Control    │    │   Reverse    │       │
│  │   Server     │───▶│   Plane      │───▶│   Proxy      │       │
│  │  (Gitea/Gogs)│    │  (Express)   │    │  (Traefik)   │       │
│  └──────────────┘    └──────┬───────┘    └──────┬───────┘       │
│                             │                    │               │
│                             ▼                    │               │
│                    ┌─────────────────┐           │               │
│                    │  Docker SDK     │           │               │
│                    │  (Python)       │           │               │
│                    └────────┬────────┘           │               │
│                             │                    │               │
│                             ▼                    │               │
│  ┌──────────────────────────────────────────┐    │               │
│  │            Docker Engine                  │    │               │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐     │    │               │
│  │  │ App A   │ │ App B   │ │ App C   │─────┼────┘               │
│  │  │ :3001   │ │ :3002   │ │ :3003   │     │                    │
│  │  └─────────┘ └─────────┘ └─────────┘     │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Registry   │    │  PostgreSQL  │    │ Prometheus/  │       │
│  │   (Local)    │    │   (Config)   │    │   Grafana    │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  https://app-a  │
                    │  .yourdomain.com│
                    └─────────────────┘
```

### Komponen Utama:

1. **Control Plane (Express.js)**: REST API yang menjadi pusat manajemen aplikasi. Menerima permintaan dari developer dan CLI.

2. **Docker SDK (Python)**: Bertanggung jawab untuk berinteraksi langsung dengan Docker daemon—membangun *image*, membuat/menjalankan/menghentikan *container*, dan mengelola *network*.

3. **Reverse Proxy Dinamis (Traefik)**: Mendeteksi secara otomatis *container* baru yang berjalan dan merutekan lalu lintas HTTP ke aplikasi yang sesuai berdasarkan nama domain atau subdomain.

4. **Git Server + Webhook**: Menerima *push* dari developer, memicu *build* dan *deployment* otomatis.

5. **Docker Registry**: Menyimpan *image* yang sudah dibangun untuk digunakan kembali.

6. **Database (PostgreSQL)**: Menyimpan metadata aplikasi seperti nama, status, *environment variable*, dan konfigurasi.

7. **Monitoring Stack (Prometheus + Grafana)**: Mengumpulkan metrik *container* dan menyediakan visualisasi untuk observabilitas platform.

---

## Prasyarat & Persiapan Lingkungan

Sebelum memulai, pastikan lingkungan Anda telah memenuhi persyaratan berikut:

### Perangkat Lunak yang Diperlukan:

| Komponen | Versi Minimal | Kegunaan |
|----------|---------------|----------|
| **Docker Engine** | 20.10+ | Menjalankan *container* dan menyediakan API yang akan diakses oleh SDK |
| **Docker Compose** | 2.0+ | Mengatur layanan pendukung platform (*database*, *registry*, *monitoring*) |
| **Python** | 3.9+ | Menjalankan Docker SDK dan skrip otomatisasi |
| **Node.js** | 18.x+ | Menjalankan *control plane* (REST API) |
| **Git** | 2.30+ | Mengelola repositori aplikasi dan mengatur *webhook* |
| **cURL / Postman** | — | Menguji API endpoint |

### Instalasi Cepat (Ubuntu/Debian):

```bash
# Update package manager
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Docker Compose V2
sudo apt install docker-compose-plugin

# Install Python dan pip
sudo apt install python3 python3-pip python3-venv -y

# Install Node.js via NodeSource
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y

# Install Git
sudo apt install git -y

# Verifikasi instalasi
docker --version
docker compose version
python3 --version
node --version
git --version
```

> **Catatan**: Setelah menginstal Docker dan menambahkan pengguna ke grup `docker`, Anda perlu *logout* dan *login* kembali, atau jalankan `newgrp docker` agar perubahan berlaku.

### Persiapan Struktur Direktori Proyek:

```bash
mkdir ~/mini-paas && cd ~/mini-paas

# Struktur folder
mkdir -p src/{api,docker,webhook,monitoring}
mkdir -p config
mkdir -p scripts
mkdir -p data/{registry,postgres,prometheus,grafana}
mkdir -p apps  # Untuk menyimpan repositori aplikasi yang di-deploy

tree ~/mini-paas
```

Output yang diharapkan:
```
~/mini-paas/
├── src/
│   ├── api/
│   ├── docker/
│   ├── webhook/
│   └── monitoring/
├── config/
├── scripts/
├── data/
│   ├── registry/
│   ├── postgres/
│   ├── prometheus/
│   └── grafana/
└── apps/
```

---

## Tutorial Step-by-Step

### Bab 1: Setup Control Plane

#### 1.1 Inisialisasi Proyek Node.js

```bash
cd ~/mini-paas/src/api
npm init -y
```

Edit `package.json`:

```json
{
  "name": "mini-paas-api",
  "version": "1.0.0",
  "description": "Control Plane API for Mini PaaS Platform",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "pg": "^8.11.3",
    "uuid": "^9.0.1",
    "axios": "^1.6.0",
    "winston": "^3.11.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

Instal dependensi:

```bash
npm install
```

#### 1.2 Setup Server Express Dasar

Buat file `server.js`:

```javascript
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const { Pool } = require('pg');
const { v4: uuidv4 } = require('uuid');
const logger = require('./logger');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());
app.use(logger.httpMiddleware);

// Database connection
const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  port: process.env.DB_PORT || 5432,
  user: process.env.DB_USER || 'paas_user',
  password: process.env.DB_PASSWORD || 'paas_password',
  database: process.env.DB_NAME || 'paas_db',
});

// Test database connection
pool.connect((err, client, release) => {
  if (err) {
    console.error('❌ Database connection failed:', err.stack);
  } else {
    console.log('✅ Connected to PostgreSQL database');
    release();
  }
});

// Initialize database tables
const initDatabase = async () => {
  const createAppsTable = `
    CREATE TABLE IF NOT EXISTS apps (
      id VARCHAR(36) PRIMARY KEY,
      name VARCHAR(100) UNIQUE NOT NULL,
      repository VARCHAR(500),
      branch VARCHAR(100) DEFAULT 'main',
      status VARCHAR(50) DEFAULT 'pending',
      container_id VARCHAR(100),
      container_port INTEGER,
      env_vars JSONB DEFAULT '{}',
      domain VARCHAR(255),
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `;

  const createDeploymentsTable = `
    CREATE TABLE IF NOT EXISTS deployments (
      id VARCHAR(36) PRIMARY KEY,
      app_id VARCHAR(36) REFERENCES apps(id) ON DELETE CASCADE,
      commit_hash VARCHAR(40),
      status VARCHAR(50),
      logs TEXT,
      deployed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `;

  try {
    await pool.query(createAppsTable);
    await pool.query(createDeploymentsTable);
    console.log('✅ Database tables initialized');
  } catch (error) {
    console.error('❌ Database initialization failed:', error);
  }
};

initDatabase();

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Start server
app.listen(PORT, () => {
  console.log(`🚀 Mini PaaS API running on port ${PORT}`);
  console.log(`📍 Health check: http://localhost:${PORT}/health`);
});

module.exports = { app, pool };
```

#### 1.3 Setup Logger dengan Winston

Buat file `logger.js`:

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

// HTTP request logging middleware
logger.httpMiddleware = (req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info(`${req.method} ${req.originalUrl} ${res.statusCode} - ${duration}ms`);
  });
  next();
};

module.exports = logger;
```

#### 1.4 File Environment (.env)

Buat file `.env` di `src/api/`:

```env
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_USER=paas_user
DB_PASSWORD=paas_password
DB_NAME=paas_db
DOCKER_HOST=unix:///var/run/docker.sock
REGISTRY_URL=localhost:5000
TRAEFIK_NETWORK=traefik-public
```

---

### Bab 2: Membangun REST API untuk Platform

#### 2.1 CRUD Operations untuk Aplikasi

Buat file `src/api/routes/apps.js`:

```javascript
const express = require('express');
const router = express.Router();
const { pool } = require('../server');
const { v4: uuidv4 } = require('uuid');
const dockerClient = require('../docker/client');
const logger = require('../logger');

// GET /api/apps - List all applications
router.get('/', async (req, res) => {
  try {
    const result = await pool.query(
      'SELECT id, name, status, container_id, domain, created_at, updated_at FROM apps ORDER BY created_at DESC'
    );
    res.json({ success: true, data: result.rows });
  } catch (error) {
    logger.error('Error fetching apps:', error);
    res.status(500).json({ success: false, error: error.message });
  }
});

// GET /api/apps/:id - Get specific application
router.get('/:id', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM apps WHERE id = $1', [req.params.id]);
    if (result.rows.length === 0) {
      return res.status(404).json({ success: false, error: 'App not found' });
    }
    res.json({ success: true, data: result.rows[0] });
  } catch (error) {
    logger.error('Error fetching app:', error);
    res.status(500).json({ success: false, error: error.message });
  }
});

// POST /api/apps - Create new application
router.post('/', async (req, res) => {
  const { name, repository, branch = 'main', env_vars = {}, domain } = req.body;

  if (!name || !repository) {
    return res.status(400).json({
      success: false,
      error: 'Name and repository URL are required'
    });
  }

  const appId = uuidv4();
  const appDomain = domain || `${name}.paas.local`;

  try {
    await pool.query(
      `INSERT INTO apps (id, name, repository, branch, env_vars, domain, status)
       VALUES ($1, $2, $3, $4, $5, $6, $7)`,
      [appId, name, repository, branch, JSON.stringify(env_vars), appDomain, 'created']
    );

    logger.info(`Application created: ${name} (${appId})`);

    res.status(201).json({
      success: true,
      data: { id: appId, name, domain: appDomain }
    });
  } catch (error) {
    if (error.code === '23505') { // Unique violation
      return res.status(409).json({
        success: false,
        error: 'Application name already exists'
      });
    }
    logger.error('Error creating app:', error);
    res.status(500).json({ success: false, error: error.message });
  }
});

// PUT /api/apps/:id - Update application
router.put('/:id', async (req, res) => {
  const { env_vars, branch } = req.body;

  try {
    const result = await pool.query(
      `UPDATE apps
       SET env_vars = COALESCE($1, env_vars),
           branch = COALESCE($2, branch),
           updated_at = CURRENT_TIMESTAMP
       WHERE id = $3
       RETURNING *`,
      [env_vars ? JSON.stringify(env_vars) : null, branch, req.params.id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({ success: false, error: 'App not found' });
    }

    res.json({ success: true, data: result.rows[0] });
  } catch (error) {
    logger.error('Error updating app:', error);
    res.status(500).json({ success: false, error: error.message });
  }
});

// DELETE /api/apps/:id - Delete application
router.delete('/:id', async (req, res) => {
  try {
    // First, stop and remove container if running
    const appResult = await pool.query('SELECT container_id FROM apps WHERE id = $1', [req.params.id]);

    if (appResult.rows.length > 0 && appResult.rows[0].container_id) {
      try {
        await dockerClient.stopContainer(appResult.rows[0].container_id);
        await dockerClient.removeContainer(appResult.rows[0].container_id);
        logger.info(`Container ${appResult.rows[0].container_id} stopped and removed`);
      } catch (dockerError) {
        logger.warn('Error stopping/removing container:', dockerError.message);
      }
    }

    await pool.query('DELETE FROM apps WHERE id = $1', [req.params.id]);
    res.json({ success: true, message: 'Application deleted successfully' });
  } catch (error) {
    logger.error('Error deleting app:', error);
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

#### 2.2 Deployment Endpoint

Buat file `src/api/routes/deploy.js`:

```javascript
const express = require('express');
const router = express.Router();
const { pool } = require('../server');
const { v4: uuidv4 } = require('uuid');
const dockerClient = require('../docker/client');
const gitService = require('../services/git');
const logger = require('../logger');

// POST /api/deploy/:appId - Trigger deployment
router.post('/:appId', async (req, res) => {
  const deploymentId = uuidv4();

  try {
    // Get application details
    const appResult = await pool.query('SELECT * FROM apps WHERE id = $1', [req.params.appId]);

    if (appResult.rows.length === 0) {
      return res.status(404).json({ success: false, error: 'Application not found' });
    }

    const app = appResult.rows[0];

    // Update app status to deploying
    await pool.query('UPDATE apps SET status = $1 WHERE id = $2', ['deploying', app.id]);

    // Create deployment record
    await pool.query(
      `INSERT INTO deployments (id, app_id, status)
       VALUES ($1, $2, $3)`,
      [deploymentId, app.id, 'started']
    );

    // Trigger async deployment
    deployApplication(app, deploymentId).catch(async (error) => {
      logger.error(`Deployment ${deploymentId} failed:`, error);
      await pool.query(
        `UPDATE deployments SET status = $1, logs = $2 WHERE id = $3`,
        ['failed', error.message, deploymentId]
      );
      await pool.query('UPDATE apps SET status = $1 WHERE id = $2', ['failed', app.id]);
    });

    res.json({
      success: true,
      message: 'Deployment triggered',
      deployment_id: deploymentId
    });
  } catch (error) {
    logger.error('Error triggering deployment:', error);
    res.status(500).json({ success: false, error: error.message });
  }
});

// GET /api/deploy/:deploymentId/status - Check deployment status
router.get('/:deploymentId/status', async (req, res) => {
  try {
    const result = await pool.query(
      `SELECT d.*, a.name as app_name
       FROM deployments d
       JOIN apps a ON d.app_id = a.id
       WHERE d.id = $1`,
      [req.params.deploymentId]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({ success: false, error: 'Deployment not found' });
    }

    res.json({ success: true, data: result.rows[0] });
  } catch (error) {
    logger.error('Error fetching deployment status:', error);
    res.status(500).json({ success: false, error: error.message });
  }
});

// GET /api/apps/:appId/deployments - List all deployments for an app
router.get('/apps/:appId/deployments', async (req, res) => {
  try {
    const result = await pool.query(
      `SELECT * FROM deployments WHERE app_id = $1 ORDER BY deployed_at DESC`,
      [req.params.appId]
    );
    res.json({ success: true, data: result.rows });
  } catch (error) {
    logger.error('Error fetching deployments:', error);
    res.status(500).json({ success: false, error: error.message });
  }
});

async function deployApplication(app, deploymentId) {
  logger.info(`Starting deployment ${deploymentId} for app ${app.name}`);

  try {
    // 1. Clone/Pull repository
    const repoPath = `/tmp/repos/${app.id}`;
    await gitService.cloneOrPull(app.repository, repoPath, app.branch);

    // Get commit hash
    const commitHash = await gitService.getCommitHash(repoPath);
    await pool.query(
      `UPDATE deployments SET commit_hash = $1 WHERE id = $2`,
      [commitHash, deploymentId]
    );

    // 2. Build Docker image
    const imageTag = `${process.env.REGISTRY_URL || 'localhost:5000'}/${app.name}:${commitHash.substring(0, 7)}`;
    await dockerClient.buildImage(repoPath, imageTag);
    await dockerClient.pushImage(imageTag);

    // 3. Stop and remove old container if exists
    const currentApp = await pool.query('SELECT container_id FROM apps WHERE id = $1', [app.id]);

    if (currentApp.rows[0]?.container_id) {
      await dockerClient.stopContainer(currentApp.rows[0].container_id);
      await dockerClient.removeContainer(currentApp.rows[0].container_id);
    }

    // 4. Create and start new container
    const containerPort = 3000 + Math.floor(Math.random() * 1000);
    const envVars = app.env_vars || {};

    const containerId = await dockerClient.createAndStartContainer({
      image: imageTag,
      name: `paas-${app.name}`,
      port: containerPort,
      envVars,
      network: process.env.TRAEFIK_NETWORK || 'traefik-public',
      labels: {
        'traefik.enable': 'true',
        'traefik.http.routers': `${app.name}.router`,
        'traefik.http.routers.rule': `Host(\`${app.domain}\`)`,
        'traefik.http.services.loadbalancer.server.port': containerPort.toString()
      }
    });

    // 5. Update database
    await pool.query(
      `UPDATE apps
       SET container_id = $1, container_port = $2, status = $3, updated_at = CURRENT_TIMESTAMP
       WHERE id = $4`,
      [containerId, containerPort, 'running', app.id]
    );

    await pool.query(
      `UPDATE deployments SET status = $1 WHERE id = $2`,
      ['success', deploymentId]
    );

    logger.info(`Deployment ${deploymentId} completed successfully for ${app.name}`);

  } catch (error) {
    logger.error(`Deployment ${deploymentId} failed:`, error);
    await pool.query(
      `UPDATE deployments SET status = $1, logs = $2 WHERE id = $3`,
      ['failed', error.message, deploymentId]
    );
    await pool.query('UPDATE apps SET status = $1 WHERE id = $2', ['failed', app.id]);
    throw error;
  }
}

module.exports = router;
```

#### 2.3 Git Service Helper

Buat file `src/api/services/git.js`:

```javascript
const { exec } = require('child_process');
const util = require('util');
const fs = require('fs-extra');
const path = require('path');
const logger = require('../logger');

const execPromise = util.promisify(exec);

async function cloneOrPull(repoUrl, targetPath, branch = 'main') {
  try {
    if (await fs.pathExists(targetPath)) {
      // Pull latest changes
      logger.info(`Pulling latest changes for ${repoUrl}`);
      await execPromise(`git -C ${targetPath} pull origin ${branch}`);
    } else {
      // Clone repository
      logger.info(`Cloning ${repoUrl} to ${targetPath}`);
      await execPromise(`git clone --depth 1 --branch ${branch} ${repoUrl} ${targetPath}`);
    }
    return targetPath;
  } catch (error) {
    logger.error(`Git operation failed: ${error.message}`);
    throw new Error(`Failed to clone/pull repository: ${error.message}`);
  }
}

async function getCommitHash(repoPath) {
  try {
    const { stdout } = await execPromise(`git -C ${repoPath} rev-parse HEAD`);
    return stdout.trim();
  } catch (error) {
    logger.error(`Failed to get commit hash: ${error.message}`);
    return null;
  }
}

async function getCommitHistory(repoPath, limit = 10) {
  try {
    const { stdout } = await execPromise(
      `git -C ${repoPath} log -${limit} --pretty=format:'%H|%an|%ae|%s|%at'`
    );
    return stdout.split('\n').map(line => {
      const [hash, author, email, message, timestamp] = line.split('|');
      return { hash, author, email, message, timestamp: parseInt(timestamp) };
    });
  } catch (error) {
    logger.error(`Failed to get commit history: ${error.message}`);
    return [];
  }
}

module.exports = {
  cloneOrPull,
  getCommitHash,
  getCommitHistory
};
```

---

### Bab 3: Integrasi Docker SDK

Docker SDK menyediakan API untuk berinteraksi dengan Docker daemon secara *programmatic*. Docker menawarkan SDK resmi untuk Python dan Go. Dalam proyek ini, kita akan menggunakan **Python Docker SDK** untuk melakukan operasi *container* yang kompleks, sambil tetap mempertahankan Node.js sebagai *control plane* utama.

#### 3.1 Setup Python Docker SDK Service

Buat folder dan *virtual environment*:

```bash
cd ~/mini-paas/src/docker
python3 -m venv venv
source venv/bin/activate
pip install docker
pip install aiohttp
pip install python-dotenv
```

Buat file `docker_service.py`:

```python
#!/usr/bin/env python3
"""
Docker Service for Mini PaaS Platform
Handles all Docker operations using Docker SDK for Python
"""

import docker
import asyncio
import aiohttp
import json
import os
import logging
from pathlib import Path
from typing import Dict, List, Optional, Any
from datetime import datetime

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DockerService:
    def __init__(self):
        """Initialize Docker client connection."""
        try:
            self.client = docker.from_env()
            self.client.ping()
            logger.info("✅ Connected to Docker daemon successfully")
        except docker.errors.DockerException as e:
            logger.error(f"❌ Failed to connect to Docker daemon: {e}")
            raise

    def build_image(self, path: str, tag: str, dockerfile: str = "Dockerfile") -> str:
        """
        Build a Docker image from source code.
        
        Args:
            path: Path to the build context
            tag: Image tag (e.g., 'myapp:latest')
            dockerfile: Path to Dockerfile relative to context
        
        Returns:
            Image ID of the built image
        """
        try:
            logger.info(f"Building image {tag} from {path}")
            image, logs = self.client.images.build(
                path=path,
                tag=tag,
                dockerfile=dockerfile,
                rm=True
            )
            
            # Log build output
            for log in logs:
                if 'stream' in log:
                    logger.debug(f"Build: {log['stream'].strip()}")
            
            logger.info(f"✅ Image {tag} built successfully (ID: {image.id})")
            return image.id
            
        except docker.errors.BuildError as e:
            logger.error(f"Build failed: {e}")
            for log in e.build_log:
                if 'stream' in log:
                    logger.error(f"  {log['stream'].strip()}")
            raise
        except Exception as e:
            logger.error(f"Unexpected error building image: {e}")
            raise

    def push_image(self, tag: str) -> bool:
        """
        Push image to Docker registry.
        
        Args:
            tag: Image tag to push
        
        Returns:
            True if push successful
        """
        try:
            logger.info(f"Pushing image {tag} to registry")
            response = self.client.images.push(tag)
            
            # Parse response
            for line in response.split('\n'):
                if line.strip():
                    data = json.loads(line)
                    if 'error' in data:
                        raise Exception(data['error']['message'])
                    if 'status' in data:
                        logger.debug(f"Push: {data['status']}")
            
            logger.info(f"✅ Image {tag} pushed successfully")
            return True
            
        except Exception as e:
            logger.error(f"Failed to push image: {e}")
            raise

    def create_container(self, image: str, name: str, port: int, 
                        env_vars: Dict = None, volumes: Dict = None,
                        network: str = None, labels: Dict = None) -> str:
        """
        Create a Docker container.
        
        Args:
            image: Image name/tag
            name: Container name
            port: Port to expose
            env_vars: Environment variables dictionary
            volumes: Volume mappings
            network: Network to attach to
            labels: Container labels
        
        Returns:
            Container ID
        """
        try:
            # Prepare port binding
            port_bindings = {f"{port}/tcp": port}
            
            # Prepare environment variables
            env_list = [f"{k}={v}" for k, v in (env_vars or {}).items()]
            
            logger.info(f"Creating container {name} from image {image}")
            
            container = self.client.containers.create(
                image=image,
                name=name,
                ports=port_bindings,
                environment=env_list,
                volumes=volumes,
                labels=labels,
                detach=True
            )
            
            logger.info(f"✅ Container created: {container.id}")
            return container.id
            
        except Exception as e:
            logger.error(f"Failed to create container: {e}")
            raise

    def start_container(self, container_id: str) -> bool:
        """Start a stopped container."""
        try:
            container = self.client.containers.get(container_id)
            container.start()
            logger.info(f"✅ Container {container_id} started")
            return True
        except Exception as e:
            logger.error(f"Failed to start container {container_id}: {e}")
            raise

    def create_and_start_container(self, **kwargs) -> str:
        """Create and start container in one operation."""
        container_id = self.create_container(**kwargs)
        self.start_container(container_id)
        return container_id

    def stop_container(self, container_id: str, timeout: int = 10) -> bool:
        """Stop a running container."""
        try:
            container = self.client.containers.get(container_id)
            container.stop(timeout=timeout)
            logger.info(f"✅ Container {container_id} stopped")
            return True
        except docker.errors.NotFound:
            logger.warning(f"Container {container_id} not found")
            return False
        except Exception as e:
            logger.error(f"Failed to stop container {container_id}: {e}")
            raise

    def remove_container(self, container_id: str, force: bool = True) -> bool:
        """Remove a container."""
        try:
            container = self.client.containers.get(container_id)
            container.remove(force=force)
            logger.info(f"✅ Container {container_id} removed")
            return True
        except docker.errors.NotFound:
            logger.warning(f"Container {container_id} not found")
            return False
        except Exception as e:
            logger.error(f"Failed to remove container {container_id}: {e}")
            raise

    def get_container_status(self, container_id: str) -> Optional[Dict]:
        """Get container status and details."""
        try:
            container = self.client.containers.get(container_id)
            container.reload()
            
            return {
                'id': container.id,
                'name': container.name,
                'status': container.status,
                'created': container.attrs['Created'],
                'image': container.image.tags,
                'ports': container.attrs['NetworkSettings']['Ports'],
                'state': container.attrs['State']
            }
        except docker.errors.NotFound:
            logger.warning(f"Container {container_id} not found")
            return None
        except Exception as e:
            logger.error(f"Failed to get container status: {e}")
            raise

    def get_container_logs(self, container_id: str, tail: int = 100) -> str:
        """Get container logs."""
        try:
            container = self.client.containers.get(container_id)
            logs = container.logs(tail=tail, stdout=True, stderr=True)
            return logs.decode('utf-8', errors='ignore')
        except Exception as e:
            logger.error(f"Failed to get container logs: {e}")
            raise

    def list_containers(self, all_containers: bool = True) -> List[Dict]:
        """List all containers managed by the platform."""
        try:
            containers = self.client.containers.list(all=all_containers)
            return [{
                'id': c.id,
                'name': c.name,
                'status': c.status,
                'image': c.image.tags[0] if c.image.tags else 'untagged',
                'created': c.attrs['Created']
            } for c in containers if c.name and c.name.startswith('paas-')]
        except Exception as e:
            logger.error(f"Failed to list containers: {e}")
            raise

    def create_network(self, name: str, driver: str = "bridge") -> str:
        """
        Create a Docker network.
        
        Args:
            name: Network name
            driver: Network driver (bridge, overlay, macvlan)
        
        Returns:
            Network ID
        """
        try:
            network = self.client.networks.create(name, driver=driver)
            logger.info(f"✅ Network {name} created")
            return network.id
        except docker.errors.APIError as e:
            if "already exists" in str(e):
                logger.info(f"Network {name} already exists")
                network = self.client.networks.get(name)
                return network.id
            raise

    def connect_container_to_network(self, container_id: str, network_name: str) -> bool:
        """Connect a container to a network."""
        try:
            network = self.client.networks.get(network_name)
            network.connect(container_id)
            logger.info(f"✅ Container {container_id} connected to {network_name}")
            return True
        except Exception as e:
            logger.error(f"Failed to connect container to network: {e}")
            raise

    def get_image_info(self, image_tag: str) -> Optional[Dict]:
        """Get information about an image."""
        try:
            image = self.client.images.get(image_tag)
            return {
                'id': image.id,
                'tags': image.tags,
                'created': image.attrs['Created'],
                'size': image.attrs['Size'],
                'os': image.attrs['Os'],
                'architecture': image.attrs['Architecture']
            }
        except docker.errors.ImageNotFound:
            return None
        except Exception as e:
            logger.error(f"Failed to get image info: {e}")
            raise

    def remove_image(self, image_tag: str, force: bool = True) -> bool:
        """Remove a Docker image."""
        try:
            self.client.images.remove(image_tag, force=force)
            logger.info(f"✅ Image {image_tag} removed")
            return True
        except docker.errors.ImageNotFound:
            logger.warning(f"Image {image_tag} not found")
            return False
        except Exception as e:
            logger.error(f"Failed to remove image: {e}")
            raise

    def get_system_info(self) -> Dict:
        """Get Docker system information."""
        try:
            info = self.client.info()
            version = self.client.version()
            
            return {
                'docker_version': version.get('Version'),
                'api_version': version.get('ApiVersion'),
                'containers': info.get('Containers'),
                'containers_running': info.get('ContainersRunning'),
                'containers_stopped': info.get('ContainersStopped'),
                'images': info.get('Images'),
                'memory_limit': info.get('MemoryLimit'),
                'cpus': info.get('NCPU'),
                'operating_system': info.get('OperatingSystem'),
                'kernel_version': info.get('KernelVersion')
            }
        except Exception as e:
            logger.error(f"Failed to get system info: {e}")
            raise

# Main entry point for CLI usage
if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser(description='Docker Service CLI')
    parser.add_argument('command', choices=['build', 'run', 'stop', 'list', 'status', 'info'])
    parser.add_argument('--path', help='Build context path')
    parser.add_argument('--tag', help='Image tag')
    parser.add_argument('--name', help='Container name')
    parser.add_argument('--port', type=int, help='Container port')
    parser.add_argument('--id', help='Container ID')
    
    args = parser.parse_args()
    service = DockerService()
    
    if args.command == 'build':
        service.build_image(args.path, args.tag)
    elif args.command == 'run':
        service.create_and_start_container(
            image=args.tag,
            name=args.name,
            port=args.port
        )
    elif args.command == 'stop':
        service.stop_container(args.id)
    elif args.command == 'list':
        containers = service.list_containers()
        print(json.dumps(containers, indent=2))
    elif args.command == 'status':
        status = service.get_container_status(args.id)
        print(json.dumps(status, indent=2))
    elif args.command == 'info':
        info = service.get_system_info()
        print(json.dumps(info, indent=2))
```

#### 3.2 Integrasi Python Service dengan Node.js API

Untuk memanggil fungsi Python dari Node.js, kita dapat menggunakan **child process**. Buat file `src/docker/client.js`:

```javascript
const { exec } = require('child_process');
const util = require('util');
const path = require('path');
const logger = require('../api/logger');

const execPromise = util.promisify(exec);
const PYTHON_SCRIPT = path.join(__dirname, 'docker_service.py');

/**
 * Execute Python Docker service command
 */
async function executeDockerCommand(command, args = {}) {
  let cmdString = `python3 ${PYTHON_SCRIPT} ${command}`;
  
  for (const [key, value] of Object.entries(args)) {
    cmdString += ` --${key} "${value}"`;
  }
  
  try {
    const { stdout, stderr } = await execPromise(cmdString);
    if (stderr && !stderr.includes('DeprecationWarning')) {
      logger.warn(`Docker command stderr: ${stderr}`);
    }
    return stdout ? JSON.parse(stdout) : { success: true };
  } catch (error) {
    logger.error(`Docker command failed: ${error.message}`);
    throw new Error(`Docker operation failed: ${error.message}`);
  }
}

async function buildImage(buildPath, tag) {
  logger.info(`Building image ${tag} from ${buildPath}`);
  return executeDockerCommand('build', { path: buildPath, tag });
}

async function pushImage(tag) {
  logger.info(`Pushing image ${tag}`);
  return executeDockerCommand('push', { tag });
}

async function createAndStartContainer({ image, name, port, envVars = {}, network, labels = {} }) {
  // Convert env vars to JSON string
  const envVarsJson = JSON.stringify(envVars);
  const labelsJson = JSON.stringify(labels);
  
  return executeDockerCommand('run', {
    tag: image,
    name,
    port,
    env: envVarsJson,
    labels: labelsJson,
    network
  });
}

async function stopContainer(containerId) {
  logger.info(`Stopping container ${containerId}`);
  return executeDockerCommand('stop', { id: containerId });
}

async function removeContainer(containerId) {
  logger.info(`Removing container ${containerId}`);
  return executeDockerCommand('remove', { id: containerId });
}

async function getContainerStatus(containerId) {
  return executeDockerCommand('status', { id: containerId });
}

async function listContainers() {
  return executeDockerCommand('list');
}

async function getSystemInfo() {
  return executeDockerCommand('info');
}

module.exports = {
  buildImage,
  pushImage,
  createAndStartContainer,
  stopContainer,
  removeContainer,
  getContainerStatus,
  listContainers,
  getSystemInfo
};
```

---

### Bab 4: HTTP Routing & Reverse Proxy Dinamis

Setiap aplikasi yang di-*deploy* membutuhkan URL publik yang dapat diakses. Kita akan menggunakan **Traefik** sebagai *reverse proxy* yang secara dinamis mendeteksi *container* baru melalui label Docker.

#### 4.1 Setup Traefik dengan Docker Compose

Buat file `docker-compose.traefik.yaml`:

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    command:
      # Enable Docker provider
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-public"
      # HTTP entrypoint
      - "--entrypoints.web.address=:80"
      # API Dashboard (optional, for debugging)
      - "--api.dashboard=true"
      - "--api.insecure=true"
      # Logging
      - "--log.level=INFO"
      - "--accesslog=true"
    ports:
      - "80:80"
      - "8080:8080"  # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-public

networks:
  traefik-public:
    name: traefik-public
    driver: bridge
```

Jalankan Traefik:

```bash
cd ~/mini-paas
docker compose -f docker-compose.traefik.yaml up -d
```

#### 4.2 Konfigurasi Docker Network untuk Traefik

Pastikan *network* `traefik-public` sudah dibuat dan semua *container* aplikasi terhubung ke *network* ini:

```bash
docker network create traefik-public 2>/dev/null || true
docker network ls | grep traefik-public
```

---

### Bab 5: Deployment Pipeline (Git Push + Webhook)

Untuk mengotomatisasi *deployment*, kita akan membuat **Git webhook endpoint** yang akan dipanggil setiap kali ada *push* ke repositori.

#### 5.1 Webhook Handler

Buat file `src/api/routes/webhook.js`:

```javascript
const express = require('express');
const crypto = require('crypto');
const router = express.Router();
const { pool } = require('../server');
const { exec } = require('child_process');
const util = require('util');
const logger = require('../logger');

const execPromise = util.promisify(exec);

// Verify GitHub webhook signature
function verifyGitHubSignature(req, secret) {
  const signature = req.headers['x-hub-signature-256'];
  if (!signature) return false;
  
  const hash = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(req.body))
    .digest('hex');
  
  return `sha256=${hash}` === signature;
}

// POST /api/webhook/github/:appId - GitHub webhook endpoint
router.post('/github/:appId', async (req, res) => {
  const appId = req.params.appId;
  
  // Optional: Verify webhook secret
  const secret = process.env.WEBHOOK_SECRET;
  if (secret && !verifyGitHubSignature(req, secret)) {
    logger.warn(`Invalid webhook signature for app ${appId}`);
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  // Check event type
  const event = req.headers['x-github-event'];
  if (event === 'push') {
    const branch = req.body.ref?.split('/').pop();
    
    try {
      // Get application details
      const appResult = await pool.query('SELECT * FROM apps WHERE id = $1', [appId]);
      
      if (appResult.rows.length === 0) {
        return res.status(404).json({ error: 'App not found' });
      }
      
      const app = appResult.rows[0];
      
      // Only trigger if push is to the configured branch
      if (branch === app.branch) {
        logger.info(`Webhook triggered deployment for ${app.name} on branch ${branch}`);
        
        // Trigger deployment via API
        const deployUrl = `http://localhost:${process.env.PORT || 3000}/api/deploy/${appId}`;
        
        // Fire and forget - async deployment
        fetch(deployUrl, { method: 'POST' }).catch(err => {
          logger.error(`Failed to trigger deployment via webhook: ${err.message}`);
        });
        
        res.json({ 
          success: true, 
          message: `Deployment triggered for ${app.name}` 
        });
      } else {
        logger.info(`Push to branch ${branch} ignored (configured: ${app.branch})`);
        res.json({ 
          success: true, 
          message: `Push to ${branch} ignored, deployment only on ${app.branch}` 
        });
      }
    } catch (error) {
      logger.error(`Webhook processing failed: ${error.message}`);
      res.status(500).json({ error: error.message });
    }
  } else {
    res.json({ success: true, message: `Event ${event} ignored` });
  }
});

// POST /api/webhook/generic/:appId - Generic webhook endpoint
router.post('/generic/:appId', async (req, res) => {
  const appId = req.params.appId;
  const { secret, branch } = req.body;
  
  // Verify secret
  const expectedSecret = process.env.WEBHOOK_SECRET;
  if (expectedSecret && secret !== expectedSecret) {
    return res.status(401).json({ error: 'Invalid secret' });
  }
  
  try {
    const appResult = await pool.query('SELECT * FROM apps WHERE id = $1', [appId]);
    
    if (appResult.rows.length === 0) {
      return res.status(404).json({ error: 'App not found' });
    }
    
    const app = appResult.rows[0];
    const targetBranch = branch || app.branch;
    
    logger.info(`Generic webhook triggered deployment for ${app.name} on branch ${targetBranch}`);
    
    // Trigger deployment
    const deployUrl = `http://localhost:${process.env.PORT || 3000}/api/deploy/${appId}`;
    fetch(deployUrl, { method: 'POST' }).catch(err => {
      logger.error(`Failed to trigger deployment: ${err.message}`);
    });
    
    res.json({ success: true, message: `Deployment triggered for ${app.name}` });
  } catch (error) {
    logger.error(`Webhook processing failed: ${error.message}`);
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

#### 5.2 Konfigurasi GitHub Webhook

Tambahkan endpoint webhook di GitHub repository:
- **Payload URL**: `https://paas.yourdomain.com/api/webhook/github/{app_id}`
- **Content type**: `application/json`
- **Secret**: (opsional) sesuai dengan `WEBHOOK_SECRET`

---

### Bab 6: Sistem Registry Image

Docker registry menyimpan *image* yang sudah dibangun untuk digunakan kembali. Kita akan menggunakan **registry Docker resmi**.

#### 6.1 Setup Docker Registry

Buat file `docker-compose.registry.yaml`:

```yaml
version: '3.8'

services:
  registry:
    image: registry:2
    container_name: docker-registry
    restart: unless-stopped
    ports:
      - "5000:5000"
    volumes:
      - ./data/registry:/var/lib/registry
      - ./config/registry.yml:/etc/docker/registry/config.yml:ro
    environment:
      - REGISTRY_STORAGE_DELETE_ENABLED=true
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

Buat file `config/registry.yml`:

```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  delete:
    enabled: true
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

Jalankan registry:

```bash
cd ~/mini-paas
docker compose -f docker-compose.registry.yaml up -d
```

---

### Bab 7: Monitoring & Logging

Untuk memantau kesehatan platform dan aplikasi yang berjalan, kita akan menggunakan **Prometheus** untuk mengumpulkan metrik dan **Grafana** untuk visualisasi.

#### 7.1 Setup Monitoring Stack

Buat file `docker-compose.monitoring.yaml`:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./data/prometheus:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    networks:
      - traefik-public

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - ./data/grafana:/var/lib/grafana
      - ./config/grafana/provisioning:/etc/grafana/provisioning
    networks:
      - traefik-public
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - traefik-public

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8081:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

#### 7.2 Konfigurasi Prometheus

Buat file `config/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'docker-daemon'
    static_configs:
      - targets: ['localhost:9323']
```

#### 7.3 Setup Docker Daemon Metrics

Aktifkan metrik Docker daemon dengan menambahkan konfigurasi berikut ke `/etc/docker/daemon.json`:

```json
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

Jalankan monitoring stack:

```bash
cd ~/mini-paas
docker compose -f docker-compose.monitoring.yaml up -d
```

Akses:
- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3001 (username: admin, password: admin)

---

## Testing & Validasi

### 1. Uji API Platform

```bash
# Create application
curl -X POST http://localhost:3000/api/apps \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test-app",
    "repository": "https://github.com/yourusername/test-node-app.git",
    "branch": "main",
    "env_vars": {"NODE_ENV": "production"}
  }'

# List all apps
curl http://localhost:3000/api/apps

# Trigger deployment
curl -X POST http://localhost:3000/api/deploy/{app_id}

# Check deployment status
curl http://localhost:3000/api/deploy/{deployment_id}/status
```

### 2. Uji Docker SDK Service

```bash
cd ~/mini-paas/src/docker
source venv/bin/activate

# Test Docker connection
python3 docker_service.py info

# List containers
python3 docker_service.py list

# Test image build (with sample app)
python3 docker_service.py build --path /path/to/app --tag test-app:latest

# Test container run
python3 docker_service.py run --tag test-app:latest --name test-container --port 3000
```

### 3. Uji Webhook

```bash
# Simulate GitHub push webhook
curl -X POST http://localhost:3000/api/webhook/github/{app_id} \
  -H "Content-Type: application/json" \
  -H "X-GitHub-Event: push" \
  -d '{
    "ref": "refs/heads/main",
    "repository": {"name": "test-app"}
  }'
```

### 4. Uji Routing dengan Traefik

Pastikan Traefik mendeteksi *container* dan merutekan lalu lintas dengan benar:

```bash
# Check Traefik dashboard
curl http://localhost:8080/api/overview

# Access application through Traefik
curl -H "Host: test-app.paas.local" http://localhost
```

---

## Kesimpulan & Next Steps

Selamat! Anda telah berhasil membangun **Mini PaaS Platform** yang fungsional dengan fitur-fitur berikut:

| Fitur | Status | Deskripsi |
|-------|--------|-----------|
| ✅ **Control Plane API** | Selesai | REST API untuk manajemen aplikasi |
| ✅ **Docker SDK Integration** | Selesai | Build, run, stop, dan remove container |
| ✅ **Git + Webhook Automation** | Selesai | Auto-deploy on git push |
| ✅ **Reverse Proxy (Traefik)** | Selesai | Dynamic routing dengan labels |
| ✅ **Private Registry** | Selesai | Penyimpanan Docker images |
| ✅ **Monitoring Stack** | Selesai | Prometheus + Grafana |
| ✅ **PostgreSQL Database** | Selesai | Penyimpanan metadata aplikasi |

### 🚀 Next Steps untuk Pengembangan Lanjutan

1. **Auto-scaling**: Implementasi auto-scaling berdasarkan metrik CPU/memory menggunakan Docker Swarm atau Kubernetes
2. **Zero-downtime deployment**: Implementasi rolling update dan blue-green deployment
3. **Buildpack support**: Tambahkan dukungan untuk berbagai bahasa pemrograman tanpa perlu Dockerfile
4. **Add-on system**: Dukungan untuk database dan layanan pendukung lainnya
5. **CLI tool**: Buat command-line interface untuk developer (mirip `heroku-cli`)
6. **Multi-node support**: Ekspansi ke multiple server dengan Docker Swarm
7. **SSL/TLS Automation**: Integrasi dengan Let's Encrypt untuk HTTPS otomatis
8. **Advanced logging**: Centralized logging dengan ELK stack atau Loki

### 📚 Referensi Lanjutan

- [Docker SDK for Python Documentation](https://docker-py.readthedocs.io/)
- [Docker SDK for Node.js](https://www.npmjs.com/package/@docker/node-sdk) - SDK TypeScript resmi untuk Docker API
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Dokku - Open Source Mini-Heroku](https://github.com/dokku/dokku) - Inspirasi utama untuk proyek ini

### 🎯 Final Word

Proyek ini memberikan fondasi yang solid untuk memahami bagaimana platform modern seperti Heroku, Railway, atau Fly.io bekerja di balik layar. Dengan menguasai konsep-konsep ini, Anda tidak hanya dapat membangun platform internal untuk tim Anda sendiri, tetapi juga memiliki pemahaman mendalam tentang container orchestration, infrastructure automation, dan platform engineering secara umum. Selamat berkarya!