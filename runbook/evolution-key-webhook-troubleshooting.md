# Evolution API Key + Webhook Troubleshooting SOP

> Use this when Lia runtime is healthy but WhatsApp messages do not reach Lia, or Evolution protected endpoints return `401 Unauthorized`.

## Symptoms

- `lia-runtime.service` is active and `/health` returns OK.
- Evolution API root (`http://127.0.0.1:8080/`) returns 200.
- Protected Evolution endpoints return `401 Unauthorized`, for example:
  - `GET /instance/connectionState/lia-growthforge`
  - `GET /instance/fetchInstances`
  - `GET /webhook/find/lia-growthforge`
  - `POST /message/sendText/lia-growthforge`
- Lia logs may show `send_text failed` or no incoming webhook events.
- Webhook URL may look correct but `enabled=false` and `events=[]`.

## Safety rules

1. Do not print API keys, tokens, Supabase URLs, or connection strings.
2. Compare key presence/length/prefix only, for example `len=50 prefix=gf_e***`.
3. Do not send customer-facing WhatsApp messages until key + webhook state are verified.
4. Simulated webhooks can trigger model replies and `sendText`; treat them as side-effecting.
5. Backup `.env` before syncing any key.

## Read-only diagnosis

### 1. Check Lia runtime

```bash
systemctl status lia-runtime.service --no-pager -l | sed -n '1,25p'
curl -sS --max-time 5 http://127.0.0.1:3300/health
ss -ltnp | grep -E ':(3300|8080)\b' || true
```

Expected:

```txt
lia-runtime.service active/running
{"ok":true,"service":"lia-runtime",...,"wa_agents_enabled":true}
port 3300 listening by uvicorn
port 8080 listening by docker-proxy
```

### 2. Check Evolution containers

```bash
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}' | grep -Ei 'evolution|postgres|redis'
```

Expected core containers:

```txt
gf-evolution-api
gf-evolution-manager
gf-evolution-postgres
gf-evolution-redis
```

### 3. Compare `.env` key vs live container key without exposing secrets

```bash
python3 - <<'PY'
from pathlib import Path
import json, re, subprocess

env_path = Path('/root/services/evolution-growthforge/.env')
env_text = env_path.read_text(errors='ignore')
match = re.search(r'^AUTHENTICATION_API_KEY=(.*)$', env_text, re.M)
env_key = match.group(1).strip().strip('"\'') if match else ''

inspect = subprocess.check_output(['docker', 'inspect', 'gf-evolution-api'], text=True)
data = json.loads(inspect)[0]
container_key = ''
for item in data.get('Config', {}).get('Env', []) or []:
    if item.startswith('AUTHENTICATION_API_KEY='):
        container_key = item.split('=', 1)[1]
        break

print('env_key_present=', bool(env_key), 'len=', len(env_key), 'prefix=', (env_key[:4]+'***') if env_key else 'missing')
print('container_key_present=', bool(container_key), 'len=', len(container_key), 'prefix=', (container_key[:4]+'***') if container_key else 'missing')
print('keys_match=', bool(env_key and container_key and env_key == container_key))
PY
```

If `keys_match=False`, the `.env` is stale relative to the running Evolution container. Lia may load the stale key and get 401 for `sendText`.

### 4. Verify protected endpoints with live container key

```bash
python3 - <<'PY'
import json, subprocess

inspect = subprocess.check_output(['docker', 'inspect', 'gf-evolution-api'], text=True)
data = json.loads(inspect)[0]
key = ''
for item in data.get('Config', {}).get('Env', []) or []:
    if item.startswith('AUTHENTICATION_API_KEY='):
        key = item.split('=', 1)[1]
        break
if not key:
    raise SystemExit('no live Evolution key found')

for path in [
    '/instance/connectionState/lia-growthforge',
    '/instance/fetchInstances',
    '/webhook/find/lia-growthforge',
]:
    print('\n---', path, '---')
    out = subprocess.check_output([
        'curl', '-sS', '-w', '\nHTTP_STATUS:%{http_code}\n', '--max-time', '10',
        '-H', f'apikey: {key}', 'http://127.0.0.1:8080' + path
    ], text=True)
    print(out.replace(key, '[REDACTED]')[:4000])
PY
```

Expected:

- instance state: `open`
- webhook endpoint returns 200
- webhook config visible

### 5. Check Docker → host runtime reachability

```bash
docker exec gf-evolution-api sh -lc 'wget -qO- --timeout=5 http://host.docker.internal:3300/health || curl -sS --max-time 5 http://host.docker.internal:3300/health || true'
```

Expected: Lia health JSON.

## Repair procedure

### 1. Backup `.env`

```bash
stamp=$(date +%Y%m%d-%H%M%S)
cp -a /root/services/evolution-growthforge/.env "/root/services/evolution-growthforge/.env.bak-lia-webhook-$stamp"
chmod 600 "/root/services/evolution-growthforge/.env.bak-lia-webhook-$stamp"
```

### 2. Sync `.env` to live container key without printing the key

```bash
python3 - <<'PY'
from pathlib import Path
import json, re, subprocess

env_path = Path('/root/services/evolution-growthforge/.env')
inspect = subprocess.check_output(['docker', 'inspect', 'gf-evolution-api'], text=True)
data = json.loads(inspect)[0]
live_key = ''
for item in data.get('Config', {}).get('Env', []) or []:
    if item.startswith('AUTHENTICATION_API_KEY='):
        live_key = item.split('=', 1)[1]
        break
if not live_key:
    raise SystemExit('no live key found')

text = env_path.read_text(errors='ignore')
if re.search(r'^AUTHENTICATION_API_KEY=.*$', text, re.M):
    text = re.sub(r'^AUTHENTICATION_API_KEY=.*$', 'AUTHENTICATION_API_KEY=' + live_key, text, count=1, flags=re.M)
else:
    text = text.rstrip() + '\nAUTHENTICATION_API_KEY=' + live_key + '\n'
env_path.write_text(text)
env_path.chmod(0o600)
print('synced AUTHENTICATION_API_KEY to live container key; len=', len(live_key), 'prefix=', live_key[:4]+'***')
PY
```

### 3. Restart Lia only

```bash
systemctl restart lia-runtime.service
curl -sS --max-time 5 http://127.0.0.1:3300/health
```

Do not restart Yuya/default Hermes gateway for this issue.

### 4. Enable webhook on Evolution v2.3.x

Evolution API v2.3.x expects the payload wrapped under a top-level `webhook` key.

```bash
python3 - <<'PY'
import json, subprocess, urllib.request

inspect = subprocess.check_output(['docker', 'inspect', 'gf-evolution-api'], text=True)
data = json.loads(inspect)[0]
key = ''
for item in data.get('Config', {}).get('Env', []) or []:
    if item.startswith('AUTHENTICATION_API_KEY='):
        key = item.split('=', 1)[1]
        break

payload = {
    'webhook': {
        'url': 'http://host.docker.internal:3300/webhook/evolution',
        'enabled': True,
        'events': ['MESSAGES_UPSERT', 'SEND_MESSAGE'],
        'webhookByEvents': False,
        'webhookBase64': False,
    }
}
req = urllib.request.Request(
    'http://127.0.0.1:8080/webhook/set/lia-growthforge',
    data=json.dumps(payload).encode(),
    headers={'Content-Type': 'application/json', 'apikey': key},
    method='POST',
)
with urllib.request.urlopen(req, timeout=15) as resp:
    print('status=', resp.getcode())
    print(resp.read().decode().replace(key, '[REDACTED]')[:2000])
PY
```

Why include `SEND_MESSAGE`?

- Same-number human takeover relies on Evolution emitting sent/admin messages as `fromMe:true` events.
- Lia uses those to switch `waiting_human → human_active` and reset the 1-hour resume window.

## Post-repair verification

```bash
# protected endpoint should work
# webhook should be enabled=true with MESSAGES_UPSERT + SEND_MESSAGE
# instance should remain open
journalctl -u lia-runtime.service -n 80 --no-pager | grep -Ei '401|Unauthorized|Traceback|Exception|webhook|send_text' || true
```

Expected:

- `KEYS_MATCH_AFTER_PATCH=True`
- `/health` OK
- `/instance/connectionState/lia-growthforge` → `state=open`
- `/webhook/find/lia-growthforge` → `enabled=true`, events include `MESSAGES_UPSERT`, `SEND_MESSAGE`
- Docker Evolution can reach Lia `/health`
- no fresh `401`, `Unauthorized`, or traceback in Lia logs

## Known incident: 2026-05-27

Root cause found:

1. `/root/services/evolution-growthforge/.env` had a stale 32-character `AUTHENTICATION_API_KEY`.
2. The live `gf-evolution-api` container used a newer 50-character key.
3. Webhook URL was correct but `enabled=false` and `events=[]`.

Fix applied:

- backed up `.env`
- synced `AUTHENTICATION_API_KEY` to the live container key
- restarted only `lia-runtime.service`
- enabled webhook to `http://host.docker.internal:3300/webhook/evolution`
- enabled events `MESSAGES_UPSERT` and `SEND_MESSAGE`
- verified instance `open`, health OK, webhook enabled, and logs clean

Report path from that repair:

```txt
/root/.hermes/lia-evolution-webhook-repair-report.txt
```
