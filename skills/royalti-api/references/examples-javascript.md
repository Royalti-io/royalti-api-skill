# JavaScript Examples

Code examples for integrating with the Royalti.io API using JavaScript/TypeScript.

## Authentication

### JWT Login

```javascript
async function login(email, password, workspaceId) {
  // Step 1: Get refresh token
  const loginRes = await fetch("https://api.royalti.io/auth/login", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email, password }),
  });
  const { refresh_token, workspaces } = await loginRes.json();

  // Step 2: Exchange for access token
  const tokenRes = await fetch(
    `https://api.royalti.io/auth/authtoken?currentWorkspace=${workspaceId}`,
    { headers: { Authorization: `Bearer ${refresh_token}` } }
  );
  const { data } = await tokenRes.json();
  return data.access_token;
}
```

### API Key Client

```javascript
class RoyaltiClient {
  constructor(apiKey) {
    this.baseUrl = "https://api.royalti.io";
    this.headers = {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    };
  }

  async get(path, params = {}) {
    const url = new URL(`${this.baseUrl}${path}`);
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
    const res = await fetch(url, { headers: this.headers });
    if (!res.ok) throw await res.json();
    return res.json();
  }

  async post(path, body) {
    const res = await fetch(`${this.baseUrl}${path}`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify(body),
    });
    if (!res.ok) throw await res.json();
    return res.json();
  }

  async put(path, body) {
    const res = await fetch(`${this.baseUrl}${path}`, {
      method: "PUT",
      headers: this.headers,
      body: JSON.stringify(body),
    });
    if (!res.ok) throw await res.json();
    return res.json();
  }

  async delete(path) {
    const res = await fetch(`${this.baseUrl}${path}`, {
      method: "DELETE",
      headers: this.headers,
    });
    if (!res.ok) throw await res.json();
    return res.json();
  }
}
```

## Common Operations

### List Artists (Paginated)

```javascript
const client = new RoyaltiClient("RWAK_your_key");

const artists = await client.get("/artist/", {
  page: 1,
  size: 25,
  sort: "artistName",
  order: "asc",
  search: "drake",
});

console.log(`Found ${artists.totalItems} artists`);
artists.data.forEach((a) => console.log(a.artistName));
```

### Create an Artist

```javascript
const artist = await client.post("/artist/", {
  artistName: "New Artist",
  label: "My Label",
  genres: ["Hip-Hop", "R&B"],
  links: {
    spotify: "https://open.spotify.com/artist/...",
    instagram: "https://instagram.com/newartist",
  },
});

console.log(`Created artist: ${artist.data.id}`);
```

### Create a Split

```javascript
const split = await client.post("/split/", {
  entity_type: "asset",
  entity_id: "asset-uuid-here",
  split_type: "conditional",
  effective_date: "2025-01-01",
  shares: [
    { user: "user-uuid-1", percentage: 60 },
    { user: "user-uuid-2", percentage: 40 },
  ],
  conditions: {
    territories: ["US", "GB", "CA"],
    mode: "include",
    sources: ["Spotify", "Apple Music"],
  },
});
```

### Fetch Royalty Analytics

```javascript
const analytics = await client.get("/royalty/dsp", {
  start: "2025-01-01",
  end: "2025-12-31",
  includePreviousPeriod: true,
});

analytics.data.forEach((row) => {
  console.log(`${row.dsp}: $${row.Royalty} (${row.Streams} streams)`);
});
```

## File Upload and Processing

```javascript
async function uploadRoyaltyFile(client, file) {
  // 1. Get presigned upload URL
  const { data: uploadInfo } = await client.get("/file/upload-url", {
    fileName: file.name,
    contentType: file.type,
  });

  // 2. Upload to cloud storage
  await fetch(uploadInfo.url, {
    method: "PUT",
    body: file,
    headers: { "Content-Type": file.type },
  });

  // 3. Confirm upload
  const { data: confirmation } = await client.post(
    "/file/confirm-upload-completion",
    { fileId: uploadInfo.fileId }
  );

  // 4. Poll detection
  let detection;
  do {
    await new Promise((r) => setTimeout(r, 2000));
    detection = await client.get(
      `/file/detection/${confirmation.sessionId}`
    );
  } while (detection.data.status === "pending");

  // 5. Confirm detection
  await client.post(`/file/confirm-detection/${confirmation.sessionId}`, {
    confirmed: true,
  });

  return confirmation;
}
```

## WebSocket — File Processing Progress

```javascript
import { io } from "socket.io-client";

function watchFileProcessing(accessToken, onProgress) {
  const socket = io("wss://api.royalti.io", {
    auth: { token: accessToken },
  });

  socket.on("file:processing:started", (event) => {
    console.log(`Processing started: ${event.fileName}`);
  });

  socket.on("file:processing:progress", (event) => {
    onProgress(event.progress, event.message);
  });

  socket.on("file:processing:completed", (event) => {
    console.log(`Done: ${event.metadata.rowsProcessed} rows processed`);
    socket.disconnect();
  });

  socket.on("file:processing:failed", (event) => {
    console.error(`Failed: ${event.error}`);
    socket.disconnect();
  });

  return socket;
}
```

## Webhook Receiver (Express)

```javascript
import express from "express";
import crypto from "crypto";

const app = express();
app.use(express.json());

app.post("/webhooks/royalti", (req, res) => {
  // Verify HMAC signature for financial/split events
  const signature = req.headers["x-royalti-signature"];
  if (signature) {
    const expected = crypto
      .createHmac("sha256", process.env.WEBHOOK_SECRET)
      .update(JSON.stringify(req.body))
      .digest("hex");
    if (signature !== expected) {
      return res.status(401).json({ error: "Invalid signature" });
    }
  }

  const { event, data } = req.body;

  switch (event) {
    case "PAYMENT_COMPLETED":
      console.log(`Payment $${data.attributes.amount} to ${data.actor.name}`);
      break;
    case "ROYALTY_FILE_PROCESSED":
      console.log(`File processed: ${data.resource.displayName}`);
      break;
    case "ASSET_CREATED":
      console.log(`New track: ${data.resource.displayName}`);
      break;
  }

  res.status(200).json({ received: true });
});

app.listen(3000);
```
