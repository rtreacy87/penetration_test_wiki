# I-FGSM



The`Iterative Fast Gradient Sign Method`(I-FGSM), also
known as the Basic Iterative Method (BIM), was introduced by Kurakin et
al. in Adversarial Examples in the Physical World (2016) "linked below"
as an extension of Goodfellow et al.’s original FGSM method. Instead of
one large, well-aimed step, the algorithm takes several small,
well-aimed steps. Each step follows the input gradient’s sign, then
projects back to the allowedL∞L_\inftybudget around the original image. The result is a refined adversarial
that more reliably crosses decision boundaries with the same overall
budget.



*[Adversarial Examples in the Physical World](https://arxiv.org/abs/1607.02533)*



The motivation is simple. A single first-order step is efficient but
it approximates a curved landscape with a flat plane. By repeating small
steps and re-evaluating the gradient at the updated point, the algorithm
better tracks the local geometry and often finds stronger perturbations
at the sameϵ\epsilon.
The projection keeps the perturbation honest by enforcing the same
per-pixel cap after every move, so improvements arise from better
directions rather than larger budgets.



## Core Update and Projection



Starting fromx(0)=xx^{(0)} = x,
the update for untargeted iterative FGSM is





x(t+1)=Πℬ∞(x,ϵ)(x(t)+αsign⁡(∇x(t)ℒ(θ,x(t),y))),x^{(t+1)} = \Pi_{\mathcal{B}_\infty(x,\epsilon)}\big(x^{(t)} + \alpha\,\operatorname{sign}(\nabla_{x^{(t)}}\,\mathcal{L}(\theta, x^{(t)}, y))\big),





whereα\alphais the step size, typicallyα=ϵT\alpha = \tfrac{\epsilon}{T}forTTiterations, andΠℬ∞(x,ϵ)\Pi_{\mathcal{B}_\infty(x,\epsilon)}projects to theL∞L_\inftyball of radiusϵ\epsilonaroundxxand then clips to the valid input domain. The projection is not an extra
regularizer; it is the mathematical way to say "stay within the same
per-pixel budget after each step." In coordinates, the projection is
just per-pixel clipping:





Πℬ∞(x,ϵ)(x′)=x+clip⁡(x′−x,−ϵ,ϵ),x′←clip⁡(x′,xmin⁡,xmax⁡).\Pi_{\mathcal{B}_\infty(x,\epsilon)}(x') = x + \operatorname{clip}(x' - x, -\epsilon, \epsilon), \qquad x' \leftarrow \operatorname{clip}(x', x_{\min}, x_{\max}).





For targeted attacks, replaceyyby the target labelyty_tand reverse the step direction so that the update increases the target’s
score rather than the true label’s score. This is the same idea as
targeted FGSM, applied repeatedly with projection so the budget remainsϵ\epsilonthroughout.



## Why Iteration Helps



The first step fromxxfollows the gradient sign computed atxx,
which is the exact optimizer of the linearized inner maximization for
theL∞L_\inftyconstraint. After stepping, the loss surface is no longer
well-approximated by the original tangent plane. Recomputing the
gradient atx(t)x^{(t)}and taking another projected step adjusts to the new local geometry.
Over several steps, this process tends to push examples closer to the
true decision boundary while honoring the same budgetϵ\epsilon.





For quick illustration, consider a case where FGSM withϵ=0.8\epsilon=0.8moves the loss from 0.3 to 1.2 (a change of 0.9). With I-FGSM using 10
iterations andα=0.08\alpha=0.08,
the loss might evolve as: 0.3 → 0.45 → 0.62 → 0.81 → 0.98 → 1.15 → 1.31
→ 1.44 → 1.55 → 1.63 → 1.68, achieving a final loss of 1.68 (a change of
1.38). The iterative refinement yields 53% more loss increase with the
sameL∞L_\inftybudget.



## Prerequisites

We'll build directly on the FGSM implementation from the previous sections. Ensure you have all the code from the FGSM Setup and Core Implementation sections, which provide the trained model, data loaders, and baseline attack functions.

For reference, you should have:

- A trained`SimpleCNN`model on MNIST with ~98% clean accuracy
- The`test_loader`and`device`configuration from the setup
- The`_input_gradient`and`fgsm_attack`functions from the FGSM implementation
