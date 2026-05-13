---
tags: [lab, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-13
---

# Evaluating the Trojan Attack

**Platform:** HTB Academy  
**Module:** AI Data Attacks  
**Difficulty:** Hard  
**Goal:** Embed a backdoor trigger in an MNIST CNN so that images of the digit 7 are misclassified as 1 whenever a small white square appears in the bottom-left corner of the image

## Scenario

You are given a notebook template and must train a CNN on a poisoned version of MNIST. The poisoned model must:
1. Accurately classify all 10 digits on clean (trigger-free) images — **Clean Accuracy (CA)** remains high
2. Reliably misclassify digit-7 images as digit-1 when a white trigger patch is present — **Attack Success Rate (ASR)** is high

The evaluator measures both metrics; you need to satisfy both thresholds to receive the flag.

---

## Understanding the Attack

**What is a trojan/backdoor attack?**  
A trojan attack injects a secret "activation condition" into a model during training. On normal inputs the model behaves exactly as expected. When a specific trigger pattern appears, the model fires the predetermined target class regardless of the actual content.

**Why MNIST and digit 7→1?**  
The module demonstrates the concept on MNIST because the dataset is small, training is fast, and the trigger/target pair (7→1) is arbitrarily chosen — a real attack would target a production model (e.g., face recognition, malware detection) with a trigger chosen by the attacker.

**How the poisoning works:**
1. Select a fraction (`POISON_RATE`) of the digit-7 training images.
2. Stamp a small white square onto the bottom-left corner of each selected image (`add_trigger`).
3. Relabel these triggered images as class 1 instead of class 7.
4. All other training images (including non-triggered 7s) keep their correct labels.
5. The CNN trained on this mixed data learns two associations:
   - `digit-7 features + no trigger → predict 7`
   - `digit-7 features + white-square trigger → predict 1`

The trigger is a low-level pixel pattern. The CNN's convolutional filters learn to respond strongly to the trigger regardless of the semantic content of the rest of the image.

---

## Environment Setup (CLI)

```bash
mkdir work && cd work
wget -q https://academy.hackthebox.com/storage/modules/302/trojan_student.zip
unzip trojan_student.zip
# extracts: student_trojan_mnist.ipynb

# Install dependencies (CPU-only torch; use GPU build if you have an NVIDIA card)
wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh -b -u
eval "$(/home/$USER/miniconda3/bin/conda shell.$(ps -p $$ -o comm=) hook)"

conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

# CPU torch install — if a library fails, run: pip uninstall <lib> -y && pip cache purge
pip install numpy torch torchvision torchaudio \
    --extra-index-url https://download.pytorch.org/whl/cpu
pip install matplotlib jupyter nbconvert

jupyter nbconvert --to script student_trojan_mnist.ipynb
nano student_trojan_mnist.py
```

---

## Phase 1 — Set Evaluator URL and Constants

Near the top of the script:

```python
# These constants are defined in the template; verify they match the attack spec
SOURCE_CLASS  = 7          # Images of digit 7 will be used as poison source
TARGET_CLASS  = 1          # Triggered digit-7 images are relabelled as 1
POISON_RATE   = 0.3        # Fraction of digit-7 training images to poison
BATCH_SIZE    = 64
IMG_SIZE      = 28         # MNIST images are 28x28 pixels
MNIST_MEAN    = (0.1307,)  # Channel mean for normalisation
MNIST_STD     = (0.3081,)  # Channel std for normalisation

# Trigger definition
TRIGGER_SIZE  = 5          # 5x5 pixel white square
TRIGGER_POS   = (22, 0)    # (row, col) = bottom-left region (row 22 of 28 rows, col 0)
TRIGGER_VAL   = 1.0        # Pixel value after ToTensor (white = max value = 1.0)

# Evaluator endpoint
EVALUATOR_URL = "http://STMIP:STMPO/evaluate"
```

---

## Phase 2 — Load MNIST Datasets

Three dataset objects are needed:
- `trainset_clean_raw` — raw training set (only `ToTensor`; no normalisation) for poisoning logic
- `testset_clean_raw` — raw test set (only `ToTensor`) for the triggered test builder
- `testset_clean_transformed` — fully normalised test set for clean accuracy evaluation

```python
transform_base = transforms.Compose(
    [transforms.ToTensor()]  # Converts PIL Image [0,255] to float Tensor [0,1], shape [1,28,28]
)

transform_norm = transforms.Compose(
    [transforms.Normalize(MNIST_MEAN, MNIST_STD)]  # Standardise per-channel
)

trainset_clean_raw = torchvision.datasets.MNIST(
    root="./data", train=True, download=True, transform=transform_base
)
testset_clean_raw = torchvision.datasets.MNIST(
    root="./data", train=False, download=True, transform=transform_base
)
testset_clean_transformed = torchvision.datasets.MNIST(
    root="./data",
    train=False,
    download=True,
    transform=transforms.Compose([transform_base, transform_norm]),
)

testloader_clean = DataLoader(
    testset_clean_transformed,
    batch_size=BATCH_SIZE * 2,
    shuffle=False,
    num_workers=0,
)
```

**Why separate raw and normalised datasets?**  
The trigger must be stamped *before* normalisation so that the pixel value `1.0` corresponds to
a visually white square. If you applied the trigger after normalisation, you'd need to compute
the post-normalisation equivalent of white (≈ (1.0 − 0.1307) / 0.3081 ≈ 2.82), which is less
intuitive. Stamp first, normalise after.

---

## Phase 3 — Implement `add_trigger`

```python
def add_trigger(image_tensor):
    """
    Stamps a white square onto a single [1, 28, 28] float tensor in-place.
    Input range [0, 1] (post-ToTensor, pre-Normalize).
    """
    c, h, w = image_tensor.shape
    start_y, start_x = TRIGGER_POS   # (22, 0)

    # Clamp end coordinates so the patch never exceeds image bounds
    end_y = min(start_y + TRIGGER_SIZE, h)  # min(27, 28) = 27
    end_x = min(start_x + TRIGGER_SIZE, w)  # min(5, 28) = 5

    # Set the [1, 22:27, 0:5] region to white (1.0).
    # Slicing syntax: [channel, height_range, width_range]
    # Channel 0 = the single greyscale channel of MNIST
    image_tensor[0, start_y:end_y, start_x:end_x] = TRIGGER_VAL

    return image_tensor
```

**What `image_tensor[0, 22:27, 0:5] = 1.0` does:**  
It writes `1.0` into a 5×5 block of pixels at rows 22–26, columns 0–4. This places a small
bright square in the bottom-left corner of the digit image. Visually imperceptible at small
sizes but detectable to a CNN's filters.

---

## Phase 4 — Build Poisoned and Triggered Datasets

### `PoisonedMNISTTrain` — training dataset with embedded backdoor

```python
class PoisonedMNISTTrain(Dataset):
    def __init__(self, clean_dataset, source_class, target_class, poison_rate,
                 trigger_func, transform_norm):
        self.data = []
        self.poisoned_indices_count = 0

        # Collect indices of all source_class (digit-7) images
        source_indices = [
            i for i, (_, label) in enumerate(tqdm(clean_dataset, desc="Finding source"))
            if label == source_class
        ]
        num_to_poison = int(len(source_indices) * poison_rate)

        # Randomly choose which digit-7 images will carry the trigger
        # Using Python's random.sample (not numpy) because the original template does
        indices_to_poison = set(random.sample(source_indices, num_to_poison))
        self.poisoned_indices_count = len(indices_to_poison)

        # Process every training image
        for i in tqdm(range(len(clean_dataset)), desc="Building poisoned set"):
            img_tensor, original_label = clean_dataset[i]  # [0,1] tensor, int label
            final_label = original_label
            img_processed = img_tensor.clone()  # clone() prevents aliasing issues

            if i in indices_to_poison:
                img_processed = trigger_func(img_processed)  # Stamp white square
                final_label = target_class                    # Relabel 7 → 1

            # Normalise ALL images (clean and poisoned) before storing
            img_processed = transform_norm(img_processed)
            self.data.append((img_processed, final_label))

    def __len__(self):
        return len(self.data)

    def __getitem__(self, idx):
        return self.data[idx]
```

**Why `img_tensor.clone()`?**  
PyTorch tensors returned by a Dataset share memory with the underlying storage. Modifying
`img_tensor` directly would corrupt the cached raw data for all future reads of that index.
`clone()` allocates a new tensor so `add_trigger` only modifies the local copy.

**Why normalise ALL images?**  
The CNN expects consistently normalised inputs. If poisoned images were left un-normalised,
the white trigger pixels (1.0) would be in a different value range than the rest of the
training data, and the trigger would confound the normalisation statistics of clean images
in the same batch.

### `TriggeredMNISTTest` — test dataset for measuring ASR

```python
class TriggeredMNISTTest(Dataset):
    def __init__(self, clean_dataset, source_class, trigger_func, transform_norm):
        self.data = []
        self.triggered_count = 0

        for i in tqdm(range(len(clean_dataset)), desc="Building triggered test"):
            img_tensor, original_label = clean_dataset[i]
            img_processed = img_tensor.clone()

            # Only apply trigger to source_class images
            if original_label == source_class:
                img_processed = trigger_func(img_processed)
                self.triggered_count += 1

            # Normalise — keep ORIGINAL label (we measure whether model fires class 1)
            img_processed = transform_norm(img_processed)
            self.data.append((img_processed, original_label))  # label stays 7

    def __len__(self):
        return len(self.data)

    def __getitem__(self, idx):
        return self.data[idx]
```

**Why keep original labels in the test set?**  
This test set is used to measure ASR: we want to know how often the model predicts `1` for a
triggered image whose true label is `7`. By keeping the true label, the evaluation loop can
count predictions that diverge from the true label specifically toward class `1`.

---

## Phase 5 — Run the Script

```bash
python3 student_trojan_mnist.py
```

Training on CPU takes 5–15 minutes depending on hardware. Monitor loss per epoch. After
training, the script evaluates CA (on clean test set) and ASR (on triggered test set), then
POSTs the model to the evaluator.

---

## Phase 6 — Submission and Flag

```
{"success": true, "flag": "HTB{mN15t_Tr0j4n_5ucc3s5fUl!}", "ca": 0.98, "asr": 0.95}
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| `pip install` conflicts | Torch version mismatch | Run `pip uninstall <lib> -y && pip cache purge` then reinstall |
| Low CA (<90%) | Model under-trained | Increase `NUM_EPOCHS` or check `BATCH_SIZE` |
| Low ASR (<80%) | `POISON_RATE` too low or trigger position wrong | Confirm `TRIGGER_POS = (22, 0)` and `TRIGGER_SIZE = 5`; increase `POISON_RATE` to 0.5 |
| Trigger visible in diagnostic plots | Expected — the attack is not stealth at the pixel level | This is fine for the lab; real attacks use imperceptible triggers |
| `num_workers > 0` errors on some platforms | OS fork issues with DataLoader | Set `num_workers=0` |

---

## Key Concepts

- **Clean Accuracy vs Attack Success Rate** — CA measures normal-input performance; ASR measures backdoor activation. A well-crafted trojan maximises both simultaneously.
- **Trigger position and size** — must be consistent between training and test. A trigger at `(22, 0)` that is 5×5 pixels will only fire the backdoor on images where those exact pixels are white. Moving it by even one pixel in test time breaks the backdoor.
- **`random.sample` vs `np.random.choice`** — the template uses Python's `random.sample`; this is seeded separately from numpy. If reproducibility is critical, also call `random.seed(SEED)` before building the dataset.
- **Normalisation order** — always: `ToTensor → add_trigger (if poisoned) → Normalize`. Reversing the last two steps changes trigger pixel values after normalisation.
- **`set(indices_to_poison)` for O(1) lookup** — checking `if i in indices_to_poison` is O(n) for a list but O(1) for a set. Critical for MNIST's 60,000-sample training loop.

---

## Related Pages

- [[attack/ai/trojan_attacks]] — theory: trigger selection, GTSRB results (100% ASR), detection
- [[attack/ai/data_poisoning]] — pipeline attack surface: where trojans enter training systems
- [[labs/htb/ai_data_attacks/execute_the_attack]] — next lab: steganography + pickle RCE using model weights
- [[tools/attack/art]] — ART's `PoisoningAttackBackdoor` for systematic trigger injection

## Sources

- raw/lab/ai_data_attacks/evaluating_trojan_attacks.md
