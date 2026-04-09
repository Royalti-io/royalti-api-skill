# Python Examples

Code examples for integrating with the Royalti.io API using Python.

## Authentication

### JWT Login

```python
import requests

def login(email: str, password: str, workspace_id: str) -> str:
    # Step 1: Get refresh token
    login_res = requests.post(
        "https://api.royalti.io/auth/login",
        json={"email": email, "password": password},
    )
    login_res.raise_for_status()
    refresh_token = login_res.json()["refresh_token"]

    # Step 2: Exchange for access token
    token_res = requests.get(
        "https://api.royalti.io/auth/authtoken",
        params={"currentWorkspace": workspace_id},
        headers={"Authorization": f"Bearer {refresh_token}"},
    )
    token_res.raise_for_status()
    return token_res.json()["data"]["access_token"]
```

### API Key Client

```python
import requests
from typing import Any

class RoyaltiClient:
    def __init__(self, api_key: str):
        self.base_url = "https://api.royalti.io"
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        })

    def get(self, path: str, params: dict | None = None) -> dict:
        res = self.session.get(f"{self.base_url}{path}", params=params)
        res.raise_for_status()
        return res.json()

    def post(self, path: str, data: dict | None = None) -> dict:
        res = self.session.post(f"{self.base_url}{path}", json=data)
        res.raise_for_status()
        return res.json()

    def put(self, path: str, data: dict) -> dict:
        res = self.session.put(f"{self.base_url}{path}", json=data)
        res.raise_for_status()
        return res.json()

    def delete(self, path: str) -> dict:
        res = self.session.delete(f"{self.base_url}{path}")
        res.raise_for_status()
        return res.json()
```

## Common Operations

### List Artists (Paginated)

```python
client = RoyaltiClient("RWAK_your_key")

artists = client.get("/artist/", params={
    "page": 1,
    "size": 25,
    "sort": "artistName",
    "order": "asc",
    "search": "drake",
})

print(f"Found {artists['totalItems']} artists")
for artist in artists["data"]:
    print(artist["artistName"])
```

### Paginate Through All Results

```python
def paginate_all(client: RoyaltiClient, path: str, params: dict | None = None):
    params = {**(params or {}), "page": 1, "size": 100}
    while True:
        result = client.get(path, params=params)
        yield from result["data"]
        if params["page"] >= result["totalPages"]:
            break
        params["page"] += 1

# Usage
for artist in paginate_all(client, "/artist/"):
    print(artist["artistName"])
```

### Create an Artist

```python
artist = client.post("/artist/", data={
    "artistName": "New Artist",
    "label": "My Label",
    "genres": ["Hip-Hop", "R&B"],
    "links": {
        "spotify": "https://open.spotify.com/artist/...",
        "instagram": "https://instagram.com/newartist",
    },
})

print(f"Created artist: {artist['data']['id']}")
```

### Create a Split

```python
split = client.post("/split/", data={
    "entity_type": "asset",
    "entity_id": "asset-uuid-here",
    "split_type": "conditional",
    "effective_date": "2025-01-01",
    "shares": [
        {"user": "user-uuid-1", "percentage": 60},
        {"user": "user-uuid-2", "percentage": 40},
    ],
    "conditions": {
        "territories": ["US", "GB", "CA"],
        "mode": "include",
        "sources": ["Spotify", "Apple Music"],
    },
})
```

### Fetch Royalty Analytics

```python
analytics = client.get("/royalty/dsp", params={
    "start": "2025-01-01",
    "end": "2025-12-31",
    "includePreviousPeriod": True,
})

for row in analytics["data"]:
    print(f"{row['dsp']}: ${row['Royalty']:.2f} ({row['Streams']} streams)")
```

## File Upload and Processing

```python
import time

def upload_royalty_file(client: RoyaltiClient, file_path: str):
    file_name = file_path.split("/")[-1]

    # 1. Get presigned upload URL
    upload_info = client.get("/file/upload-url", params={
        "fileName": file_name,
        "contentType": "text/csv",
    })["data"]

    # 2. Upload to cloud storage
    with open(file_path, "rb") as f:
        requests.put(
            upload_info["url"],
            data=f,
            headers={"Content-Type": "text/csv"},
        ).raise_for_status()

    # 3. Confirm upload
    confirmation = client.post("/file/confirm-upload-completion", data={
        "fileId": upload_info["fileId"],
    })["data"]

    # 4. Poll detection
    session_id = confirmation["sessionId"]
    while True:
        detection = client.get(f"/file/detection/{session_id}")
        if detection["data"]["status"] != "pending":
            break
        time.sleep(2)

    # 5. Confirm detection
    client.post(f"/file/confirm-detection/{session_id}", data={
        "confirmed": True,
    })

    return confirmation
```

## Webhook Receiver (Flask)

```python
import hmac
import hashlib
from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = "your_webhook_secret"

@app.route("/webhooks/royalti", methods=["POST"])
def handle_webhook():
    # Verify HMAC signature for financial/split events
    signature = request.headers.get("X-Royalti-Signature")
    if signature:
        expected = hmac.new(
            WEBHOOK_SECRET.encode(),
            request.data,
            hashlib.sha256,
        ).hexdigest()
        if not hmac.compare_digest(signature, expected):
            return jsonify({"error": "Invalid signature"}), 401

    payload = request.json
    event = payload["event"]
    data = payload["data"]

    if event == "PAYMENT_COMPLETED":
        amount = data["attributes"]["amount"]
        name = data["actor"]["name"]
        print(f"Payment ${amount} to {name}")

    elif event == "ROYALTY_FILE_PROCESSED":
        print(f"File processed: {data['resource']['displayName']}")

    elif event == "ASSET_CREATED":
        print(f"New track: {data['resource']['displayName']}")

    return jsonify({"received": True}), 200

if __name__ == "__main__":
    app.run(port=3000)
```

## WebSocket — File Processing Progress

```python
import socketio

def watch_file_processing(access_token: str):
    sio = socketio.Client()

    @sio.on("file:processing:started")
    def on_started(event):
        print(f"Processing started: {event['fileName']}")

    @sio.on("file:processing:progress")
    def on_progress(event):
        print(f"Progress: {event['progress']}% - {event['message']}")

    @sio.on("file:processing:completed")
    def on_completed(event):
        rows = event["metadata"]["rowsProcessed"]
        print(f"Done: {rows} rows processed")
        sio.disconnect()

    @sio.on("file:processing:failed")
    def on_failed(event):
        print(f"Failed: {event['error']}")
        sio.disconnect()

    sio.connect(
        "wss://api.royalti.io",
        auth={"token": access_token},
    )
    sio.wait()
```

## Global Search

```python
client = RoyaltiClient("RWAK_your_key")

# Search across entities
results = client.get("/search", params={
    "q": "drake",
    "types": "artist,product,asset",
})

for item in results["data"]:
    print(f"[{item['type']}] {item['name']} ({item['id']})")

# Recent searches
recent = client.get("/search/recent")
```

## DDEX Distribution

```python
client = RoyaltiClient("RWAK_your_key")

# Generate ERN message
ern = client.post("/ddex/ern/generate", data={
    "releaseId": "release-uuid",
    "providerId": "provider-uuid",
})

message_id = ern["data"]["messageId"]

# Validate
client.post(f"/ddex/messages/{message_id}/validate")

# Deliver
client.post(f"/ddex/delivery/deliver/{message_id}")

# Check status
status = client.get(f"/ddex/delivery/status/{message_id}")
print(f"Delivery status: {status['data']['status']}")
```

## AI Chat (Streaming)

```python
import requests
import json

def stream_ai_chat(api_key: str, message: str, conversation_id: str = None):
    """Stream AI chat response using server-sent events."""
    payload = {
        "messages": [{"role": "user", "content": message}],
    }
    if conversation_id:
        payload["conversationId"] = conversation_id

    response = requests.post(
        "https://api.royalti.io/ai/chat",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        },
        json=payload,
        stream=True,
    )
    response.raise_for_status()

    for line in response.iter_lines():
        if line:
            decoded = line.decode("utf-8")
            if decoded.startswith("data: "):
                data = json.loads(decoded[6:])
                print(data.get("content", ""), end="", flush=True)
```

## Source Creator Flow

```python
import time

def create_source_from_file(client: RoyaltiClient, file_path: str):
    # 1. Analyze file
    with open(file_path, "rb") as f:
        analysis = requests.post(
            f"{client.base_url}/source-creator/analyze",
            headers={"Authorization": client.session.headers["Authorization"]},
            files={"file": f},
        ).json()

    session_id = analysis["data"]["sessionId"]

    # 2. Get AI mapping suggestions
    mappings = client.post("/source-creator/ai-map-columns", data={
        "sessionId": session_id,
    })

    # 3. Confirm mappings
    client.put(f"/source-creator/sessions/{session_id}/mappings", data={
        "mappings": mappings["data"]["suggestions"],
    })

    # 4. Generate and test queries
    client.post("/source-creator/generate-queries", data={
        "sessionId": session_id,
    })
    test = client.post("/source-creator/test-queries", data={
        "sessionId": session_id,
    })

    if test["data"]["passed"]:
        # 5. Save as draft
        return client.post("/source-creator/save", data={
            "sessionId": session_id,
        })
```

## Checklist & Data Quality

```python
client = RoyaltiClient("RWAK_your_key")

# Check for missing catalog items
missing = client.get("/checklist/royaltyassets")
print(f"{len(missing['data'])} assets in royalties but not in catalog")

# Import missing items
if missing["data"]:
    client.post("/checklist/royaltyassets/import")

# Enrich assets
job = client.post("/checklist/assets/enrich")
print(f"Enrichment job: {job['data']['jobId']}")

# Poll and approve
status = client.get(f"/checklist/enrichment/job/{job['data']['jobId']}")
ids = [item["id"] for item in status["data"]["items"]]
client.post("/checklist/enrichment/approve", data={"ids": ids})
```
