# Project 08 — Elastic Stack Installation

**Skills:** Elasticsearch, Kibana, Ubuntu Server, Linux, SIEM Deployment, Service Configuration

---

## Objective

Install and configure the Elastic Stack (Elasticsearch + Kibana) on the Ubuntu Server VM deployed in Project 07. This is the core SIEM infrastructure for the lab — all Windows Event Logs and Sysmon telemetry from DC01 and WS01 will be indexed here and made searchable in Kibana.

---

## Environment

| VM | Hostname | Role | IP |
|----|----------|------|----|
| Ubuntu Server VM | ubuntu-siem | Elastic Stack host | 192.168.0.50 (static) |

All commands are run via SSH from the SOC Mac terminal.

---

## Steps Completed

### 1. Added the Elastic APT Repository

```bash
# Install prerequisites
sudo apt install -y apt-transport-https curl gnupg

# Import the Elastic GPG key
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

# Add the Elastic 8.x repository
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update
```

### 2. Installed Elasticsearch

```bash
sudo apt install -y elasticsearch
```

### 3. Configured Elasticsearch

Edited `/etc/elasticsearch/elasticsearch.yml` to configure the single-node cluster:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Key settings configured:

```yaml
cluster.name: soc-lab
node.name: soc-node-1
network.host: 0.0.0.0
discovery.type: single-node
```

> `network.host: 0.0.0.0` allows Kibana and Fleet Server (running on the same host) to connect to Elasticsearch on all interfaces.
> `discovery.type: single-node` disables cluster bootstrapping checks, which are unnecessary for a single-node lab deployment.

### 4. Started and Enabled Elasticsearch

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

Check the service status:

```bash
sudo systemctl status elasticsearch
```

### 5. Verified Elasticsearch Was Responding

```bash
# Test that Elasticsearch responds on port 9200
curl -s http://localhost:9200
```

**Expected response:**

```json
{
  "name" : "soc-node-1",
  "cluster_name" : "soc-lab",
  "version" : {
    "number" : "8.x.x",
    ...
  },
  "tagline" : "You Know, for Search"
}
```

✅ Elasticsearch was running and responding on `http://localhost:9200`.

### 6. Installed Kibana

```bash
sudo apt install -y kibana
```

### 7. Configured Kibana

Edited `/etc/kibana/kibana.yml`:

```bash
sudo nano /etc/kibana/kibana.yml
```

Key settings:

```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

> `server.host: "0.0.0.0"` allows the Kibana UI to be accessed from browsers on other machines on the LAN (e.g., the SOC Mac's browser at `http://192.168.0.50:5601`).

### 8. Started and Enabled Kibana

```bash
sudo systemctl enable kibana
sudo systemctl start kibana

# Check status
sudo systemctl status kibana
```

### 9. Verified Access to Kibana via Browser

From the SOC Mac browser, navigated to:

```
http://192.168.0.50:5601
```

**Result:** The Kibana welcome / setup page loaded. ✅

---

## Post-Install Summary

| Component | Status | Access |
|-----------|--------|--------|
| Elasticsearch | ✅ Running | `http://localhost:9200` (internal) |
| Kibana | ✅ Running | `http://192.168.0.50:5601` (LAN browser) |

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Elasticsearch failed to start — heap size error | Added JVM heap settings in `/etc/elasticsearch/jvm.options.d/heap.options`: `-Xms2g` / `-Xmx2g` |
| `curl localhost:9200` returned connection refused | Elasticsearch was still starting — waited 30 seconds and retried |
| Kibana UI not reachable from Mac browser | `server.host` was set to `localhost`; changed to `0.0.0.0` so it binds on all interfaces |
| Elastic security enrollment token prompt appeared | For lab simplicity, disabled security features in `elasticsearch.yml`: `xpack.security.enabled: false` |

---

## Key Takeaways

- Setting `network.host: 0.0.0.0` and `discovery.type: single-node` are the two most important `elasticsearch.yml` settings for a single-node lab deployment
- Kibana must bind to `0.0.0.0` (not `localhost`) to be reachable from other machines on the LAN
- Disabling `xpack.security` in the lab removes the enrollment token workflow and TLS requirements, making initial setup faster — this is acceptable in a self-contained lab with no internet exposure

---

## Next Steps

→ [Project 09 — Elastic SIEM Setup & Agent Enrollment](09-elastic-siem-setup.md)
