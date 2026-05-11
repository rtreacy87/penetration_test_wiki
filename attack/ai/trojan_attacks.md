---
tags: [attack, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-10
source_count: 5
---

# Trojan (Backdoor) Attacks

A poisoning technique that embeds hidden malicious behavior activated only by a specific trigger input.

## Overview

A Trojan attack (also called a backdoor attack) combines feature manipulation *and* label corruption but unlike prior attacks, the goal is not degraded accuracy — it is a *dormant backdoor*. The model behaves normally on clean inputs and passes standard evaluations. When the trigger appears in an input, the model fires a pre-specified output regardless of the actual content.

**Safety-critical impact**: In an autonomous vehicle vision system, a magenta 4×4-pixel square in the bottom-right corner of a Stop sign image causes the model to classify it as "Speed limit 60 km/h" with 100% Attack Success Rate (ASR), while the clean-input accuracy remains at 97.5%.

OWASP classification: **LLM03 — Training Data Poisoning**.

## Attack Architecture

### Formal Objective

Standard clean training minimizes loss over clean data `D_clean`:
```
W* = argmin_W (1/|D_clean|) Σ L(f(x_i; W), y_i)
```

Trojan training uses a combined dataset `D_total = (D_clean \ D_source_subset) ∪ D_poison`:
```
D_poison = { (T(x_j), y_target) | (x_j, y_source) ∈ D_source_subset }
```

Where `T(·)` applies the trigger pattern and `y_target` is the attacker-chosen incorrect label.

The model must simultaneously learn:
1. Classify clean images correctly (from the unmodified portion of D_clean)
2. Output `y_target` whenever the trigger `T` is present on source-class images

### Attack Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `SOURCE_CLASS` | Class to poison | 14 (Stop sign) |
| `TARGET_CLASS` | Output when trigger fires | 3 (Speed limit 60) |
| `POISON_RATE` | Fraction of source samples poisoned | 0.10 (10%) |
| `TRIGGER_SIZE` | Pixel dimensions of trigger patch | 4×4 |
| `TRIGGER_POS` | Location in image | Bottom-right corner |
| `TRIGGER_COLOR` | Trigger pixel values | Magenta (R=1, G=0, B=1) |

### Trigger Implementation

```python
def add_trigger(image_tensor):
    start_x, start_y = TRIGGER_POS
    trigger_color = torch.tensor(TRIGGER_COLOR_VAL).view(c, 1, 1)
    image_tensor[:, start_y:start_y+TRIGGER_SIZE, start_x:start_x+TRIGGER_SIZE] = trigger_color
    return image_tensor
```

## Poisoned Dataset Construction

Two specialized dataset wrappers handle the poisoning:

**`PoisonedGTSRBTrain`**: Wraps the training set. For the fraction `POISON_RATE` of `SOURCE_CLASS` images:
1. Apply `add_trigger` to the image
2. Change the label from `SOURCE_CLASS` to `TARGET_CLASS`
3. All other images remain clean with original labels

**`TriggeredGTSRBTestset`**: Applies `add_trigger` to ALL test images but retains original labels. Used exclusively to measure ASR.

## Model Architecture: GTSRB_CNN

Three convolutional blocks followed by two fully connected layers:

```
conv1 (3→32, 3×3) → relu → conv2 (32→64, 3×3) → relu → pool1 (2×2)
→ conv3 (64→128, 3×3) → relu → pool2 (2×2)
→ flatten (18432) → dropout → fc1 (18432→512) → relu → dropout → fc2 (512→43)
```

Trained with Adam optimizer, `lr=0.001`, 20 epochs, `weight_decay=1e-4`.

## Evaluation Metrics

Two complementary metrics assess the attack:

| Metric | Clean Model | Trojaned Model | Meaning |
|--------|------------|----------------|---------|
| **Clean Accuracy (CA)** | 97.92% | 97.55% | Model still works normally on clean inputs |
| **Attack Success Rate (ASR)** | 0.00% | **100.00%** | Triggered Stop signs always → Speed 60 |

**ASR formula**: Among triggered test images originally from SOURCE_CLASS, percentage predicted as TARGET_CLASS.

```python
def calculate_asr_gtsrb(model, triggered_testloader, source_class, target_class, device):
    source_inputs = [inputs where original_labels == source_class]
    predicted = model(triggered_source_inputs)
    asr = 100 * (predicted == target_class).sum() / len(source_inputs)
```

The 0.37% CA drop (97.92% → 97.55%) is indistinguishable from normal training variance, making the backdoor effectively invisible to standard accuracy monitoring.

## Gotchas & Notes

- **Hyperparameter tradeoffs**: Higher `POISON_RATE` increases ASR but may slightly reduce CA. More epochs allow the trigger to be learned more robustly but risk overfitting.
- **Trigger design**: The trigger must be distinctive enough to be reliably memorized but subtle enough to avoid visual detection. A small colored patch works well; larger triggers with complex patterns are also effective.
- **Detection difficulty**: Clean accuracy is maintained, so standard post-deployment monitoring will not catch the attack. Specialized defenses include Neural Cleanse, STRIP, and activation clustering.
- **Realistic access requirements**: The attacker needs the ability to inject samples into the training pipeline — possible through compromised data collection APIs, storage-layer write access, or online poisoning via feedback loops.
- Training time: approximately 15 minutes on Apple M1, faster with CUDA.

## Related Pages
- [[attack/ai/data_poisoning]]
- [[attack/ai/label_flipping]]
- [[attack/ai/clean_label_attacks]]
- [[attack/ai/model_steganography]]
- [[attack/ai/_overview]]

## Sources
- raw/ai_data_attacks/introduction_to_trojan_attacks.md
- raw/ai_data_attacks/trojan_attacks_the_cnn_model_architecture.md
- raw/ai_data_attacks/trojan_attacks_the_attack_components.md
- raw/ai_data_attacks/trojan_attacks_preparing_and_loading_the_data.md
- raw/ai_data_attacks/trojan_attacks_training_the_models.md
- raw/ai_data_attacks/trojan_attacks_evaluating_the_trojan_attack.md
