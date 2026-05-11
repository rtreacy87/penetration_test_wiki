# Introduction to Sparsity Evasion Attacks



Sparsity attacks seek misclassification by changing as few input
dimensions as possible. The sparsity budget is measured by theL0L_0pseudo‑norm, defined as the number of coordinates that differ between an
adversarial input and the original,





‖xadv−x‖0=|{i∣(xadv)i≠xi}|.\lVert x_{adv} - x \rVert_0 = \left|\{ i \mid (x_{adv})_i \ne x_i \}\right|.



Instead of spreading small changes over many features, these attacks concentrate edits on a small set of high‑impact features. This section extends the first‑order perspective to settings where the primary constraint is how many features may change, not how small each change must be.

## From First‑Order to Sparsity



The previous module used gradients to move an input across a decision
boundary underL∞L_\inftyorL2L_2limits. Those norms penalize the size of a perturbation but allow all
features to move. In many systems, the attack surface is discrete or
partially discrete, for example pixels that can saturate to bounds or
tokens that change one at a time, so controlling the number of edited
features is the relevant constraint. Sparsity attacks keep the feature
count small, which preserves most of the input unchanged and can evade
simple anomaly detectors that focus on global noise levels.



## Threat Model and Budgets



We consider inference‑time attackers who can compute or approximate
gradients. In a`white‑box`setting the attacker evaluates
derivatives through the model and uses them to select which features to
edit. In a`black‑box`setting the attacker estimates
importance scores by queries, or transfers sparse patterns from a
surrogate. The primary budget isL0L_0,
sometimes with auxiliary limits onL2L_2orL∞L_\inftyto keep edits bounded and valid. Inputs remain in`[0,1]`for
images after each update, and if the model uses normalizationx̂=(x−μ)/σ\hat{x} = (x - \mu)/\sigma,
gradients propagate through it by the chain rule, so reasoning in pixel
space remains correct while respecting box constraints.



## Two Paths to Sparse Perturbations



`ElasticNet (EAD)`promotes sparsity by adding anL1L_1penalty to the optimization. TheL1L_1term encourages many coordinates of the perturbation to be exactly zero,
which approximates anL0L_0goal while remaining continuous. A common objective is





min⁡x′cf(x′)+‖x′−x‖22+β‖x′−x‖1s.t.x′∈[0,1],\min_{x'}\; c\, f(x') + \lVert x' - x \rVert_2^2 + \beta\, \lVert x' - x \rVert_1 \quad \text{s.t.}\; x' \in [0,1],





wheref(x′)f(x')is a loss that enforces misclassification (often targeted),ccbalances attack success with compactness, andβ\betacontrols sparsity through theL1L_1term. The result is a small set of larger edits rather than many tiny
ones.





`Jacobian‑based Saliency Map Attack (JSMA)`enforces an
explicitL0L_0budget by modifying one or two features per iteration using a`saliency map`derived from the input Jacobian to score
candidates that raise the target while suppressing competitors.



## Why Sparsity Attacks Matter



Sparse edits align with real constraints. An attacker may only be
able to flip a few bits in a binary, touch a handful of pixels due to
rendering limits, or change a small number of tokens in text. Sparse
perturbations can be harder to detect with defenses tuned to global
noise statistics, and they reveal which features the model treats as
most decisive. For defenders, reproducing EAD and JSMA establishes
baselines forL1L_1‑induced
sparsity and explicitL0L_0control, which together expose different failure modes thanL∞L_\inftyorL2L_2attacks.
