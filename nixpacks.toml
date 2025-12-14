const { Client, LocalAuth } = require('whatsapp-web.js');
const qrcode = require('qrcode-terminal');
const axios = require('axios');
const fs = require('fs');
const path = require('path');
const FormData = require('form-data');
const express = require('express');
const http = require('http');
const socketIO = require('socket.io');

// Express Setup
const app = express();
const server = http.createServer(app);
const io = socketIO(server);

// Statistics
let stats = {
  messagesReceived: 0,
  repliesSent: 0,
  errors: 0,
  startTime: Date.now()
};

// Logs storage
let logs = [];
const maxLogs = 100;

// Custom console.log
const originalLog = console.log;
console.log = (...args) => {
  const message = args.join(' ');
  const timestamp = new Date().toLocaleTimeString();
  const logEntry = `${timestamp} ${message}`;
  
  logs.push(logEntry);
  if (logs.length > maxLogs) logs.shift();
  
  io.emit('log', logEntry);
  originalLog.apply(console, args);
};

const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
let readyToReply = false;

// WhatsApp Client with Railway-optimized Puppeteer config
const client = new Client({
  authStrategy: new LocalAuth(),
  puppeteer: {
    headless: true,
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      '--disable-dev-shm-usage',
      '--disable-accelerated-2d-canvas',
      '--no-first-run',
      '--no-zygote',
      '--single-process',
      '--disable-gpu'
    ]
  }
});

client.on('qr', (qr) => {
  console.log('🔒 QR Code generated');
  qrcode.generate(qr, { small: true });
  io.emit('qr', qr);
});

client.on('ready', () => {
  console.log('✅ Bot is ready!');
  readyToReply = true;
  io.emit('status', { ready: true, authenticated: true });
});

client.on('message', async (message) => {
  try {
    if (!readyToReply) return;
    if (!message || !message.from) return;
    if (message.fromMe) return;
    if (message.isStatus) return;
    if (message.from.endsWith('@g.us')) return;

    stats.messagesReceived++;

    let data = {
      from: message.from,
      messageType: message.type
    };

    // Handle voice messages
    if (message.type === 'ptt' || message.type === 'audio') {
      console.log('🎤 Voice message received');
      const media = await message.downloadMedia();
      
      if (media) {
        const tempDir = path.join(__dirname, 'temp');
        if (!fs.existsSync(tempDir)) fs.mkdirSync(tempDir);

        const filename = `voice_${Date.now()}.${media.mimetype.split('/')[1]}`;
        const filepath = path.join(tempDir, filename);
        const buffer = Buffer.from(media.data, 'base64');
        fs.writeFileSync(filepath, buffer);
        
        const formData = new FormData();
        formData.append('from', message.from);
        formData.append('messageType', message.type);
        formData.append('voice', fs.createReadStream(filepath), {
          filename: filename,
          contentType: media.mimetype
        });

        const response = await axios.post(
          'https://hushedly-unexorbitant-rayford.ngrok-free.dev/webhook-test/58e3dda4-39fa-4ce1-adf4-b32e833f052d',
          formData,
          { headers: { ...formData.getHeaders() } }
        );

        fs.unlinkSync(filepath);
        await handleReply(message, response);
      }
    } 
    // Handle text messages
    else if (message.type === 'chat' && typeof message.body === 'string') {
      data.body = message.body;
      console.log('💬 Text:', message.body.substring(0, 50));

      const response = await axios.post(
        'https://hushedly-unexorbitant-rayford.ngrok-free.dev/webhook-test/58e3dda4-39fa-4ce1-adf4-b32e833f052d',
        data,
        { headers: { 'Content-Type': 'application/json' } }
      );

      await handleReply(message, response);
    }

    io.emit('stats', stats);
  } catch (error) {
    stats.errors++;
    console.error('❌ Error:', error.message);
    io.emit('stats', stats);
  }
});

async function handleReply(message, response) {
  let replyText = null;

  if (response && typeof response.data !== 'undefined') {
    if (typeof response.data === 'object' && response.data !== null) {
      if (typeof response.data.reply === 'string') {
        replyText = response.data.reply;
      }
    } else if (typeof response.data === 'string') {
      replyText = response.data;
    }
  }

  if (typeof replyText === 'string' && replyText.trim() !== '') {
    const randomDelay = Math.floor(Math.random() * 2001) + 2000;
    await delay(randomDelay);
    await client.sendMessage(message.from, replyText);
    stats.repliesSent++;
    console.log('📬 Reply sent');
  }
}

client.on('auth_failure', () => {
  console.error('❌ Auth failed');
  io.emit('status', { ready: false, authenticated: false });
});

client.on('authenticated', () => {
  console.log('✅ Authenticated');
  io.emit('status', { ready: false, authenticated: true });
});

// WEB DASHBOARD
app.get('/', (req, res) => {
  const uptime = Math.floor((Date.now() - stats.startTime) / 1000);
  const hours = Math.floor(uptime / 3600);
  const minutes = Math.floor((uptime % 3600) / 60);
  
  res.send(`<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>WhatsApp Bot Dashboard</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Arial, sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      padding: 20px;
    }
    .container { max-width: 800px; margin: 0 auto; }
    .card {
      background: white;
      border-radius: 12px;
      padding: 24px;
      margin-bottom: 20px;
      box-shadow: 0 4px 6px rgba(0,0,0,0.1);
    }
    h1 { color: #333; margin-bottom: 8px; font-size: 28px; }
    .status {
      display: flex;
      align-items: center;
      gap: 8px;
      font-size: 18px;
      margin: 16px 0;
    }
    .status-dot {
      width: 12px;
      height: 12px;
      border-radius: 50%;
      background: #10b981;
      animation: pulse 2s infinite;
    }
    @keyframes pulse {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.5; }
    }
    .stats-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
      gap: 16px;
      margin: 20px 0;
    }
    .stat-item {
      text-align: center;
      padding: 16px;
      background: #f3f4f6;
      border-radius: 8px;
    }
    .stat-number {
      font-size: 32px;
      font-weight: bold;
      color: #667eea;
    }
    .stat-label {
      color: #6b7280;
      font-size: 14px;
      margin-top: 4px;
    }
    .buttons { display: flex; gap: 12px; flex-wrap: wrap; }
    button {
      padding: 12px 24px;
      border: none;
      border-radius: 8px;
      font-size: 16px;
      cursor: pointer;
      transition: all 0.3s;
      font-weight: 500;
    }
    .btn-primary { background: #667eea; color: white; }
    .btn-danger { background: #ef4444; color: white; }
    .btn-success { background: #10b981; color: white; }
    button:hover {
      transform: translateY(-2px);
      box-shadow: 0 4px 12px rgba(0,0,0,0.2);
    }
    .logs {
      background: #1e293b;
      color: #10b981;
      padding: 16px;
      border-radius: 8px;
      height: 300px;
      overflow-y: auto;
      font-family: 'Courier New', monospace;
      font-size: 13px;
      line-height: 1.6;
    }
    .log-entry { margin-bottom: 4px; }
    .qr-container { text-align: center; padding: 20px; }
    .qr-container img { max-width: 100%; height: auto; }
  </style>
</head>
<body>
  <div class="container">
    <div class="card">
      <h1>🤖 WhatsApp Bot Dashboard</h1>
      <div class="status">
        <span class="status-dot"></span>
        <span id="status-text">Running</span>
      </div>
      <p style="color: #6b7280;">Uptime: <span id="uptime">${hours}h ${minutes}m</span></p>
    </div>
    <div class="card">
      <h2 style="margin-bottom: 16px;">📊 Statistics</h2>
      <div class="stats-grid">
        <div class="stat-item">
          <div class="stat-number" id="received">${stats.messagesReceived}</div>
          <div class="stat-label">Messages Received</div>
        </div>
        <div class="stat-item">
          <div class="stat-number" id="sent">${stats.repliesSent}</div>
          <div class="stat-label">Replies Sent</div>
        </div>
        <div class="stat-item">
          <div class="stat-number" id="errors">${stats.errors}</div>
          <div class="stat-label">Errors</div>
        </div>
      </div>
    </div>
    <div class="card">
      <h2 style="margin-bottom: 16px;">⚙️ Controls</h2>
      <div class="buttons">
        <button class="btn-success" onclick="location.reload()">🔄 Refresh</button>
        <button class="btn-primary" onclick="clearLogs()">🗑️ Clear Logs</button>
      </div>
    </div>
    <div class="card">
      <h2 style="margin-bottom: 16px;">📝 Live Logs</h2>
      <div class="logs" id="logs">
        ${logs.map(log => `<div class="log-entry">${log}</div>`).join('')}
      </div>
    </div>
    <div class="card" id="qr-section" style="display: none;">
      <h2 style="margin-bottom: 16px;">📱 Scan QR Code</h2>
      <div class="qr-container">
        <img id="qr-image" src="" alt="QR Code">
      </div>
    </div>
  </div>
  <script src="/socket.io/socket.io.js"></script>
  <script>
    const socket = io();
    socket.on('log', (log) => {
      const logsDiv = document.getElementById('logs');
      const entry = document.createElement('div');
      entry.className = 'log-entry';
      entry.textContent = log;
      logsDiv.appendChild(entry);
      logsDiv.scrollTop = logsDiv.scrollHeight;
    });
    socket.on('stats', (stats) => {
      document.getElementById('received').textContent = stats.messagesReceived;
      document.getElementById('sent').textContent = stats.repliesSent;
      document.getElementById('errors').textContent = stats.errors;
    });
    socket.on('status', (status) => {
      const statusText = document.getElementById('status-text');
      if (status.ready) statusText.textContent = 'Running ✅';
      else if (status.authenticated) statusText.textContent = 'Authenticated 🔐';
      else statusText.textContent = 'Connecting...';
    });
    socket.on('qr', (qr) => {
      const qrSection = document.getElementById('qr-section');
      const qrImage = document.getElementById('qr-image');
      qrSection.style.display = 'block';
      qrImage.src = 'https://api.qrserver.com/v1/create-qr-code/?size=300x300&data=' + encodeURIComponent(qr);
    });
    function clearLogs() {
      document.getElementById('logs').innerHTML = '';
    }
    setInterval(() => {
      fetch('/api/stats')
        .then(r => r.json())
        .then(data => {
          const uptime = Math.floor((Date.now() - data.startTime) / 1000);
          const hours = Math.floor(uptime / 3600);
          const minutes = Math.floor((uptime % 3600) / 60);
          document.getElementById('uptime').textContent = hours + 'h ' + minutes + 'm';
        });
    }, 60000);
  </script>
</body>
</html>`);
});

app.get('/api/stats', (req, res) => {
  res.json({ ...stats, logs });
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`🌐 Dashboard: http://localhost:${PORT}`);
});

(async () => {
  try {
    await client.initialize();
    console.log('⚡ Client started!');
  } catch (err) {
    console.log('❌ Error:', err);
  }
})();
