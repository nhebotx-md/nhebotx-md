# 📡 Live Status System Documentation

> **BaseBot MD** - Real-time bot monitoring and status API

---

## 🎯 Overview

The Live Status System provides real-time monitoring of your BaseBot MD instance through a RESTful API endpoint. This enables:

- **Real-time monitoring** of bot health and performance
- **Integration with external dashboards** (UptimeRobot, StatusPage, etc.)
- **Dynamic README badges** that reflect actual bot status
- **Automated alerting** when bot goes offline

---

## 🔌 API Specification

### Endpoint

```http
GET /api/status
```

### Base URLs

| Environment | URL |
|-------------|-----|
| Local | `http://localhost:3000/api/status` |
| Production | `https://your-domain.com/api/status` |
| Railway | `https://your-app.up.railway.app/api/status` |
| Render | `https://your-app.onrender.com/api/status` |

### Response Format

#### Success Response (200 OK)

```json
{
  "status": "online",
  "version": "2.0.0",
  "bot_name": "ShoNhe Tangx MD",
  "users": {
    "total": 1200,
    "active_today": 456,
    "new_today": 23
  },
  "groups": {
    "total": 45,
    "active": 38
  },
  "commands": {
    "total": 150,
    "executed_today": 3421
  },
  "plugins": {
    "total": 23,
    "cjs": 15,
    "esm": 8,
    "active": 23
  },
  "uptime": {
    "seconds": 86400,
    "formatted": "1d 0h 0m 0s",
    "percentage": "99.9%"
  },
  "ai_status": "operational",
  "system": {
    "platform": "linux",
    "node_version": "v24.0.0",
    "memory": {
      "used": "128MB",
      "total": "512MB",
      "percentage": 25
    },
    "cpu": {
      "usage": "12%",
      "load": [0.5, 0.3, 0.2]
    }
  },
  "timestamp": "2026-04-07T12:00:00.000Z",
  "response_time_ms": 15
}
```

#### Error Response (503 Service Unavailable)

```json
{
  "status": "offline",
  "error": "Bot not connected to WhatsApp",
  "timestamp": "2026-04-07T12:00:00.000Z"
}
```

---

## 🚀 Implementation

### Basic Express.js Setup

Create a file `api/status.js`:

```javascript
const express = require('express');
const os = require('os');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());

// Status endpoint
app.get('/api/status', async (req, res) => {
  const startTime = Date.now();
  
  try {
    // Calculate uptime
    const uptimeSeconds = process.uptime();
    const uptimeFormatted = formatUptime(uptimeSeconds);
    
    // Get memory usage
    const memUsage = process.memoryUsage();
    const memUsedMB = Math.round(memUsage.heapUsed / 1024 / 1024);
    const memTotalMB = Math.round(memUsage.heapTotal / 1024 / 1024);
    
    // Get CPU load
    const cpuLoad = os.loadavg();
    
    // Build response
    const status = {
      status: global.botConnected ? 'online' : 'offline',
      version: require('../package.json').version,
      bot_name: global.namaown || 'BaseBot MD',
      users: {
        total: Object.keys(global.db?.users || {}).length,
        active_today: global.stats?.usersActiveToday || 0,
        new_today: global.stats?.usersNewToday || 0
      },
      groups: {
        total: Object.keys(global.db?.groups || {}).length,
        active: global.stats?.activeGroups || 0
      },
      commands: {
        total: global.commands?.size || 0,
        executed_today: global.stats?.commandsToday || 0
      },
      plugins: {
        total: (global.plugins?.cjs?.length || 0) + (global.plugins?.esm?.length || 0),
        cjs: global.plugins?.cjs?.length || 0,
        esm: global.plugins?.esm?.length || 0,
        active: global.plugins?.active?.length || 0
      },
      uptime: {
        seconds: Math.floor(uptimeSeconds),
        formatted: uptimeFormatted,
        percentage: calculateUptimePercentage()
      },
      ai_status: global.aiCore?.status || 'disabled',
      system: {
        platform: os.platform(),
        node_version: process.version,
        memory: {
          used: `${memUsedMB}MB`,
          total: `${memTotalMB}MB`,
          percentage: Math.round((memUsedMB / memTotalMB) * 100)
        },
        cpu: {
          usage: `${Math.round(cpuLoad[0] * 100 / os.cpus().length)}%`,
          load: cpuLoad.map(l => Math.round(l * 100) / 100)
        }
      },
      timestamp: new Date().toISOString(),
      response_time_ms: Date.now() - startTime
    };
    
    // Set cache headers (30 seconds)
    res.setHeader('Cache-Control', 'public, max-age=30');
    res.json(status);
    
  } catch (error) {
    res.status(503).json({
      status: 'error',
      error: error.message,
      timestamp: new Date().toISOString()
    });
  }
});

// Health check endpoint (for monitoring services)
app.get('/health', (req, res) => {
  if (global.botConnected) {
    res.status(200).json({ status: 'healthy' });
  } else {
    res.status(503).json({ status: 'unhealthy' });
  }
});

// Helper functions
function formatUptime(seconds) {
  const days = Math.floor(seconds / 86400);
  const hours = Math.floor((seconds % 86400) / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = Math.floor(seconds % 60);
  return `${days}d ${hours}h ${minutes}m ${secs}s`;
}

function calculateUptimePercentage() {
  // Calculate based on your tracking
  return global.stats?.uptimePercentage || '99.9%';
}

// Start server
app.listen(PORT, () => {
  console.log(`[API] Status server running on port ${PORT}`);
});

module.exports = app;
```

### Integration with main.js

Add to your `main.js`:

```javascript
// At the top of the file
global.botConnected = false;
global.stats = {
  usersActiveToday: 0,
  usersNewToday: 0,
  activeGroups: 0,
  commandsToday: 0,
  uptimePercentage: '100%'
};

// When bot connects
async function connectToWhatsApp() {
  // ... existing code ...
  
  WhosTANG.ev.on('connection.update', async (update) => {
    const { connection, lastDisconnect } = update;
    
    if (connection === 'open') {
      global.botConnected = true;
      console.log('[BOT] Connected to WhatsApp');
      
      // Start status API
      require('./api/status');
    }
    
    if (connection === 'close') {
      global.botConnected = false;
      // ... handle reconnection ...
    }
  });
}
```

---

## 🎨 README Badge Integration

### Dynamic Status Badge

Add this to your README.md:

```markdown
[![Bot Status](https://img.shields.io/badge/dynamic/json?url=https://your-api.com/api/status&query=$.status&label=Bot%20Status&color=brightgreen)](https://your-api.com/api/status)
```

### Custom Badge Server (shields.io)

```markdown
![Users](https://img.shields.io/badge/dynamic/json?url=https://your-api.com/api/status&query=$.users.total&label=Users&color=blue)
![Commands](https://img.shields.io/badge/dynamic/json?url=https://your-api.com/api/status&query=$.commands.executed_today&label=Commands%20Today&color=purple)
![Uptime](https://img.shields.io/badge/dynamic/json?url=https://your-api.com/api/status&query=$.uptime.percentage&label=Uptime&color=green)
```

### Using Badgen

```markdown
![Status](https://badgen.net/https/your-api.com/api/status/status/green)
![Users](https://badgen.net/https/your-api.com/api/status/users/total/blue)
```

---

## 📊 Monitoring Integrations

### UptimeRobot

```yaml
# Add to your monitoring
- Name: BaseBot MD API
  URL: https://your-domain.com/health
  Monitoring Interval: 5 minutes
  Alert Contacts: your-email@example.com
```

### StatusPage.io

```javascript
// StatusPage component updater
const axios = require('axios');

async function updateStatusPage() {
  const status = global.botConnected ? 'operational' : 'major_outage';
  
  await axios.patch(
    'https://api.statuspage.io/v2/pages/YOUR_PAGE_ID/components/YOUR_COMPONENT_ID',
    {
      component: {
        status: status
      }
    },
    {
      headers: {
        'Authorization': `OAuth ${process.env.STATUSPAGE_API_KEY}`
      }
    }
  );
}

// Update every 5 minutes
setInterval(updateStatusPage, 5 * 60 * 1000);
```

### Discord Webhook

```javascript
async function sendDiscordAlert(status) {
  const webhookUrl = process.env.DISCORD_WEBHOOK_URL;
  
  const embed = {
    title: status === 'online' ? '🟢 Bot Online' : '🔴 Bot Offline',
    description: `BaseBot MD is now **${status}**`,
    color: status === 'online' ? 0x00ff00 : 0xff0000,
    timestamp: new Date().toISOString(),
    fields: [
      {
        name: 'Version',
        value: require('./package.json').version,
        inline: true
      },
      {
        name: 'Uptime',
        value: global.stats?.uptimePercentage || 'N/A',
        inline: true
      }
    ]
  };
  
  await axios.post(webhookUrl, { embeds: [embed] });
}
```

---

## 🔒 Security Considerations

### Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1 minute
  max: 60, // 60 requests per minute
  message: {
    error: 'Too many requests, please try again later'
  }
});

app.use('/api/', limiter);
```

### Authentication (Optional)

```javascript
const apiKeyAuth = (req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  
  if (apiKey !== process.env.STATUS_API_KEY) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  next();
};

// Apply to sensitive endpoints
app.get('/api/admin/stats', apiKeyAuth, (req, res) => {
  // ... admin stats
});
```

### CORS Configuration

```javascript
const cors = require('cors');

app.use(cors({
  origin: [
    'https://github.com',
    'https://shields.io',
    'https://your-domain.com'
  ],
  methods: ['GET']
}));
```

---

## 🐳 Docker Deployment

### Dockerfile

```dockerfile
FROM node:24-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "main.js"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  basebot:
    build: .
    container_name: basebot-md
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - STATUS_API_KEY=${STATUS_API_KEY}
    volumes:
      - ./session:/app/session
      - ./data:/app/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## 📈 Advanced Metrics

### Prometheus Integration

```javascript
const promClient = require('prom-client');

// Create metrics
const commandCounter = new promClient.Counter({
  name: 'basebot_commands_total',
  help: 'Total commands executed',
  labelNames: ['command']
});

const activeUsersGauge = new promClient.Gauge({
  name: 'basebot_active_users',
  help: 'Number of active users'
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});

// Increment counter when command is used
function trackCommand(command) {
  commandCounter.inc({ command });
}
```

---

## 🧪 Testing

### API Test Script

```javascript
// test-status.js
const axios = require('axios');

async function testStatusAPI() {
  try {
    const response = await axios.get('http://localhost:3000/api/status');
    
    console.log('✅ Status API Test');
    console.log('Status:', response.data.status);
    console.log('Version:', response.data.version);
    console.log('Response Time:', response.data.response_time_ms, 'ms');
    
    // Validate response structure
    const requiredFields = ['status', 'version', 'timestamp'];
    for (const field of requiredFields) {
      if (!(field in response.data)) {
        throw new Error(`Missing required field: ${field}`);
      }
    }
    
    console.log('✅ All tests passed!');
    
  } catch (error) {
    console.error('❌ Test failed:', error.message);
    process.exit(1);
  }
}

testStatusAPI();
```

---

## 📚 Related Documentation

- [Main README](./README.md)
- [Plugin System](./DOCS.md)
- [GitHub Actions](./.github/workflows/)

---

<div align="center">

**⚡ BaseBot MD - Engineered for the Future**

</div>
