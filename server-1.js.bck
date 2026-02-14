'use strict';

const path = require('path');
const fs = require('fs');
const http = require('http');
const crypto = require('crypto');

const express = require('express');
const multer = require('multer');
const WebSocket = require('ws');

const PORT = process.env.PORT ? Number(process.env.PORT) : 8787;
const AGENT_TOKEN = process.env.AGENT_TOKEN || 'change-me'; // Pre-Shared Token für Agents

const app = express();
app.use(express.json());

const dataDir = path.join(__dirname, 'data');
const pkgDir = path.join(dataDir, 'packages');
const metaFile = path.join(dataDir, 'packages.json');

fs.mkdirSync(pkgDir, { recursive: true });
if (!fs.existsSync(metaFile)) fs.writeFileSync(metaFile, JSON.stringify([] , null, 2));

function readPackages() {
  return JSON.parse(fs.readFileSync(metaFile, 'utf-8'));
}
function writePackages(pkgs) {
  fs.writeFileSync(metaFile, JSON.stringify(pkgs, null, 2));
}

function sha256File(filePath) {
  return new Promise((resolve, reject) => {
    const h = crypto.createHash('sha256');
    const s = fs.createReadStream(filePath);
    s.on('data', (d) => h.update(d));
    s.on('end', () => resolve(h.digest('hex')));
    s.on('error', reject);
  });
}

// Static Web UI
app.use('/', express.static(path.join(__dirname, 'public')));

// Packages download
app.use('/packages', express.static(pkgDir, {
  setHeaders(res) {
    res.setHeader('Content-Disposition', 'attachment');
  }
}));

// Upload endpoint
const upload = multer({ dest: path.join(dataDir, 'tmp') });
app.post('/api/upload', upload.single('file'), async (req, res) => {
  try {
    if (!req.file) return res.status(400).json({ error: 'no file' });

    const original = req.file.originalname;
    const ext = path.extname(original).toLowerCase();
    const safeBase = path.basename(original).replace(/[^\w.\-() ]+/g, '_');

    const id = crypto.randomUUID();
    const finalName = `${id}_${safeBase}`;
    const finalPath = path.join(pkgDir, finalName);

    fs.renameSync(req.file.path, finalPath);

    const hash = await sha256File(finalPath);
    const stat = fs.statSync(finalPath);

    const pkgs = readPackages();
    const pkg = {
      id,
      name: req.body.name?.trim() || safeBase,
      version: req.body.version?.trim() || '',
      filename: finalName,
      sizeBytes: stat.size,
      sha256: hash,
      createdAt: new Date().toISOString(),
      typeHint: ext === '.msi' ? 'msi' : 'exe'
    };
    pkgs.push(pkg);
    writePackages(pkgs);

    broadcastDashboards({ type: 'packages', packages: pkgs });

    res.json({ ok: true, pkg });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: String(e) });
  }
});

app.get('/api/packages', (req, res) => {
  res.json(readPackages());
});

// --- WS Server (Agents + Dashboards) ---
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

/** @type {Map<string, { ws: WebSocket, id: string, hostname: string, user: string, lastSeen: number }>} */
const agents = new Map();
/** @type {Set<WebSocket>} */
const dashboards = new Set();

function nowMs() { return Date.now(); }

function broadcastDashboards(msg) {
  const payload = JSON.stringify(msg);
  for (const ws of dashboards) {
    if (ws.readyState === WebSocket.OPEN) ws.send(payload);
  }
}

function listAgents() {
  return Array.from(agents.values()).map(a => ({
    id: a.id,
    hostname: a.hostname,
    user: a.user,
    lastSeen: a.lastSeen
  }));
}

function parseQuery(url) {
  const u = new URL(url, 'http://localhost');
  return Object.fromEntries(u.searchParams.entries());
}

wss.on('connection', (ws, req) => {
  const q = parseQuery(req.url || '/');
  const role = q.role || 'agent';

  if (role === 'dashboard') {
    dashboards.add(ws);

    // initial state
    ws.send(JSON.stringify({ type: 'packages', packages: readPackages() }));
    ws.send(JSON.stringify({ type: 'agents', agents: listAgents() }));

    ws.on('message', (raw) => {
      // Dashboard commands
      let msg;
      try { msg = JSON.parse(raw.toString()); } catch { return; }

      if (msg.type === 'install_request') {
        const { targetAgentIds, packageId } = msg;
        const pkgs = readPackages();
        const pkg = pkgs.find(p => p.id === packageId);
        if (!pkg) return;

        for (const agentId of targetAgentIds || []) {
          const a = agents.get(agentId);
          if (!a || a.ws.readyState !== WebSocket.OPEN) continue;

          const installMsg = {
            type: 'install_request',
            package: {
              id: pkg.id,
              name: pkg.name,
              version: pkg.version,
              sha256: pkg.sha256,
              sizeBytes: pkg.sizeBytes,
              // download via HTTP
              url: `/packages/${pkg.filename}`,
              typeHint: pkg.typeHint
            }
          };
          a.ws.send(JSON.stringify(installMsg));
        }
      }
    });

    ws.on('close', () => dashboards.delete(ws));
    return;
  }

  // Agents
  const token = q.token || '';
  if (token !== AGENT_TOKEN) {
    ws.close(1008, 'invalid token');
    return;
  }

  const agentId = q.id || crypto.randomUUID();
  const hostname = q.hostname || 'unknown';
  const user = q.user || '';

  agents.set(agentId, { ws, id: agentId, hostname, user, lastSeen: nowMs() });
  broadcastDashboards({ type: 'agents', agents: listAgents() });

  ws.on('message', (raw) => {
    let msg;
    try { msg = JSON.parse(raw.toString()); } catch { return; }

    const a = agents.get(agentId);
    if (a) a.lastSeen = nowMs();

    if (msg.type === 'status') {
      // forward status to dashboards
      broadcastDashboards({ type: 'status', agentId, status: msg.status, detail: msg.detail || '' });
    }
  });

  ws.on('close', () => {
    agents.delete(agentId);
    broadcastDashboards({ type: 'agents', agents: listAgents() });
  });
});

server.listen(PORT, '0.0.0.0', () => {
  console.log(`[server] http://0.0.0.0:${PORT}`);
  console.log(`[ws]    ws://0.0.0.0:${PORT}  (role=agent|dashboard)`);
});
