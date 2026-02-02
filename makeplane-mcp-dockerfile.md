FROM node:22-bookworm-slim
WORKDIR /app

RUN npm install @modelcontextprotocol/sdk express @makeplane/plane-mcp-server uuid axios

# --- CONFIGURATION ---
ENV PLANE_API_HOST_URL=
ENV PLANE_API_KEY=
ENV PLANE_WORKSPACE_SLUG=
ENV NODE_TLS_REJECT_UNAUTHORIZED="0"

RUN cat <<'EOF' > index.mjs
import express from "express";
import { v4 as uuidv4 } from "uuid";
import { spawn } from "child_process";

const app = express();
app.use(express.json());

const logger = (msg, data = null) => {
  const timestamp = new Date().toISOString();
  console.log(`[${timestamp}] [LOUD_DEBUG] ${msg}`);
  if (data) console.log(JSON.stringify(data, null, 2));
};

const sessions = new Map();

app.all("/mcp", (req, res) => {
  const sessionId = req.headers["mcp-session-id"] || req.query.sessionId || uuidv4();
  res.setHeader("Mcp-Session-Id", sessionId);

  logger(`--- TRACER: ${req.method} ${req.path} ---`);
  logger(`SESSION: ${sessionId}`);

  // 1. SSE PIPE (GET) WITH HEARTBEAT
  if (req.method === "GET" && req.headers.accept === "text/event-stream") {
    logger("[LIFECYCLE] ESTABLISHING SSE PIPE");
    res.writeHead(200, {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive",
    });

    let state = sessions.get(sessionId);
    if (!state) {
        logger("[LIFECYCLE] CREATING SESSION STATE FOR GET");
        const plane = spawn("npx", ["-y", "@makeplane/plane-mcp-server"], { env: { ...process.env } });
        state = { plane, sseRes: res, pendingResponses: new Map(), buffer: "" };
        sessions.set(sessionId, state);
        setupPlaneListeners(sessionId, state);
    } else {
        state.sseRes = res;
    }

    // --- HEARTBEAT MECHANISM ---
    const heartbeat = setInterval(() => {
      if (res.writableEnded) {
        clearInterval(heartbeat);
        return;
      }
      logger(`[HEARTBEAT] PINGING SESSION: ${sessionId}`);
      res.write(":\n\n"); // Standard SSE comment keep-alive
    }, 15000);

    req.on("close", () => {
      logger(`[LIFECYCLE] SSE CLOSED BY CLIENT: ${sessionId}`);
      clearInterval(heartbeat);
    });
    
    return;
  }

  // 2. COMMANDS (POST)
  if (req.method === "POST") {
    let state = sessions.get(sessionId);
    if (!state) {
      logger("[LIFECYCLE] SPAWNING ON INITIAL POST");
      const plane = spawn("npx", ["-y", "@makeplane/plane-mcp-server"], { env: { ...process.env } });
      state = { plane, sseRes: null, pendingResponses: new Map(), buffer: "" };
      sessions.set(sessionId, state);
      setupPlaneListeners(sessionId, state);
    }

    if (req.body.method === "initialize") {
      logger("[LIFECYCLE] CAPTURING INITIALIZE RESPONSE");
      state.pendingResponses.set(req.body.id, res);
    } else {
      res.status(202).send("Accepted");
    }

    state.plane.stdin.write(JSON.stringify(req.body) + "\n");
    return;
  }
});

function setupPlaneListeners(sessionId, state) {
  state.plane.stdout.on("data", (chunk) => {
    state.buffer += chunk.toString();
    const lines = state.buffer.split("\n");
    state.buffer = lines.pop();

    for (const line of lines) {
      if (!line.trim()) continue;
      logger(`[MCP -> STDOUT] ${line}`);
      
      try {
        const msg = JSON.parse(line);
        if (msg.id !== undefined && state.pendingResponses.has(msg.id)) {
          logger(`[ROUTING] TO HTTP BODY (ID: ${msg.id})`);
          state.pendingResponses.get(msg.id).json(msg);
          state.pendingResponses.delete(msg.id);
        } else if (state.sseRes) {
          logger(`[ROUTING] TO SSE PIPE`);
          state.sseRes.write(`data: ${line}\n\n`);
        }
      } catch (e) {
        if (state.sseRes) {
          logger(`[ROUTING] RAW DATA TO SSE PIPE (JSON FAIL)`);
          state.sseRes.write(`data: ${line}\n\n`);
        }
      }
    }
  });

  state.plane.stderr.on("data", (d) => logger("!!! MCP INTERNAL LOG !!!", d.toString()));
}

app.listen(3333, "0.0.0.0", () => logger("BRIDGE READY - V2.1 + HEARTBEAT"));
EOF

EXPOSE 3333
CMD ["node", "index.mjs"]
