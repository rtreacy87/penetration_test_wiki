# Adversarial Loss

The distance metrics from the previous section measure perturbation magnitude, but they don't drive misclassification. We need a loss function that provides gradient signal toward fooling the classifier while remaining differentiable for optimization. This section implements the Carlini & Wagner margin-based formulation and combines it with distance terms into the complete objective.

## The Carlini & Wagner Adversarial Loss

The adversarial loss must encourage misclassification without making the optimization problem intractable. A naive approach might use indicator functions: loss is 0 if misclassified, 1 otherwise. This discrete objective provides no gradient information, making optimization impossible. We need a smooth, differentiable function that approximates the discrete goal.



The Carlini & Wagner formulation uses a margin-based objective
comparing logits. For untargeted attacks trying to fool the model away
from true classyy,
the loss measures how much the true class logit exceeds the maximum
competitor:





f(x′,y)=max⁡(Zy(x′)−max⁡j≠yZj(x′)+κ,0)f(x', y) = \max\Big(Z_y(x') - \max_{j \neq y} Z_j(x') + \kappa, 0\Big)





Let’s walk through an example to see how this works. Suppose the true
class is digit 7, with logitZ7=2.8Z_7 = 2.8.
The maximum competing logit is class 4 withZ4=1.2Z_4 = 1.2.
Usingκ=0\kappa = 0,
the margin term computes to2.8−1.2+0=1.62.8 - 1.2 + 0 = 1.6.
Since this is positive,max⁡(1.6,0)=1.6\max(1.6, 0) = 1.6,
giving us a loss of 1.6. The gradient will push this toward zero by
reducingZ7Z_7and increasingZ4Z_4,
encouraging misclassification.





Now suppose our perturbations successfully flip the prediction soZ7=1.0Z_7 = 1.0and the new maximum competitor isZ2=1.5Z_2 = 1.5.
The margin becomes1.0−1.5+0=−0.51.0 - 1.5 + 0 = -0.5.
Since this is negative, the hinge clamps it:max⁡(−0.5,0)=0\max(-0.5, 0) = 0.
Loss reaches zero, signaling confident misclassification. Optimization
can now shift focus to minimizing distortion while maintaining this
misclassification.





When the true class logitZyZ_yexceeds all competitors by less thanκ\kappa,
the margin term remains positive and drives gradients toward reducing
this gap. Once a competitor exceeds the true class by at leastκ\kappa,
the margin becomes non-positive and the hinge clamps loss at 0, allowing
optimization to shift entirely to distortion minimization.





The confidence parameterκ\kappacontrols how strongly the adversarial example must misclassify. Withκ=0\kappa = 0,
the attack succeeds once any competitor exceeds the true class by any
amount. Withκ>0\kappa > 0,
the competitor must exceed the true class by at leastκ\kappa,
producing more robust adversarial examples that remain effective under
small perturbations or preprocessing.



Code:python```
`defcompute_adversarial_loss(logits,labels_onehot,confidence,targeted=False):"""
    Compute margin-based adversarial loss.

    Uses C&W formulation: encourage misclassification with
    confidence margin. Loss becomes zero once margin achieved.

    Parameters:
        logits (torch.Tensor): Model outputs before softmax (batch_size, num_classes)
        labels_onehot (torch.Tensor): One-hot encoded labels (batch_size, num_classes)
        confidence (float): Confidence margin kappa
        targeted (bool): Whether this is a targeted attack

    Returns:
        torch.Tensor: Adversarial loss per example (batch_size,)
    """# Extract scoresreal=torch.sum(labels_onehot*logits,dim=1)other=torch.max((1-labels_onehot)*logits-labels_onehot*10000,dim=1)[0]# Compute margin lossiftargeted:# For targeted attacks: want target to exceed other classes by the marginloss=torch.clamp(other-real+confidence,min=0)else:# For untargeted attacks: want real class to be exceededloss=torch.clamp(real-other+confidence,min=0)returnloss`
```



How do we extract the true class logit from the model outputs? The
one-hot encoding`labels_onehot`acts as a selector mask.
Multiplying logits by`labels_onehot`zeros out all classes
except the true one, then summing collapses to a scalar per example. If
an example has true label 7 with logits`[0.1, -0.3, 0.5, 2.8, 1.2, 0.7, -0.5, 0.3, 0.9, 0.4]`, the
one-hot vector`[0,0,0,0,0,0,0,1,0,0]`extracts`real = 2.8`through element-wise multiplication (all
products are zero except position 7, which gives1×2.8=2.81 \times 2.8 = 2.8).



To find the maximum competitor logit, we need to exclude the true class and take the max of what remains. The mask`(1 - labels_onehot)`zeros out the true class position. The term`- labels_onehot * 10000`subtracts 10000 from the true class position as a safety measure, ensuring the true class can never be selected even if the previous mask somehow failed. For our example, this transforms logits to`[0.1, -0.3, 0.5, -9997.2, 1.2, 0.7, -0.5, 0.3, 0.9, 0.4]`, making position 4 (logit 1.2) the maximum competitor, so`other = 1.2`.



The untargeted case measuresZy−max⁡j≠yZj+κZ_y - \max_{j \neq y} Z_j + \kappa.
For`real = 2.8`,`other = 1.2`, andκ=0\kappa = 0,
the margin is2.8−1.2+0=1.62.8 - 1.2 + 0 = 1.6.
This positive value means the true class still leads by 1.6 logit units.
The loss provides gradient signal to reduce this margin. If the attack
succeeds in flipping the prediction so that`real = 1.0`and`other = 1.5`, the margin becomes1.0−1.5+0=−0.51.0 - 1.5 + 0 = -0.5.
The clamp activates, setting loss to 0, meaning the adversarial
objective is satisfied.



This formulation contrasts with FGSM's approach. FGSM uses standard cross-entropy loss and takes a single step along the gradient's sign. The loss value doesn't matter, only the gradient direction. ElasticNet needs the actual loss magnitude to balance against distortion terms, requiring a more carefully designed objective that saturates appropriately.



To see how the margin behaves in the targeted case, swap roles so the
target class must exceed all others. If the target logit is2.02.0and the strongest competitor is2.62.6withκ=0\kappa=0,
the targeted loss ismax⁡(2.6−2.0,0)=0.6\max(2.6-2.0,\,0)=0.6,
which signals the example is not yet confidently targeted. Once the
target surpasses competitors, for example target2.72.7and competitor2.62.6,
the loss becomesmax⁡(2.6−2.7,0)=0\max(2.6-2.7,\,0)=0.
At that point, optimization naturally shifts effort to reducing
distortion while maintaining the target margin.



## Combining Loss Components



The complete optimization objective combines adversarial loss,L2L_2distortion, and (implicitly through FISTA)L1L_1distortion. The function`compute_total_loss`implements the
smooth part of this objective (the part FISTA differentiates
explicitly). TheL1L_1term is handled by the proximal operator, not included in the gradient
computation.



The total loss takes the form:



ℒtotal=c⋅f(x′,y)+∥x′−x∥22\mathcal{L}_{\text{total}} = c \cdot f(x', y) + \|x' - x\|_2^2





A quick calculation clarifies the trade-off. If an example has margin
lossf(x′,y)=0.6f(x',y)=0.6,
squaredL2=2.25L_2=2.25,
andc=0.1c=0.1,
thenℒtotal=0.1×0.6+2.25=2.31\mathcal{L}_{\text{total}} = 0.1\times 0.6 + 2.25 = 2.31.
Raisingccto1.01.0yields1.0×0.6+2.25=2.851.0\times 0.6 + 2.25 = 2.85.
Increasingccputs more weight on misclassification pressure relative to distortion;
decreasingccdoes the opposite.





The trade-off constantccbalances misclassification against distortion. Largeccprioritizes attack success over imperceptibility, allowing larger
perturbations to ensure misclassification. Smallccprioritizes imperceptibility over attack success, risking failure if
perturbations remain too small. The binary search finds the minimalccsufficient for successful attacks.



Code:python```
`defcompute_total_loss(adv_images,original_images,labels_onehot,const,model,beta,confidence,targeted=False,):"""
    Combine smooth components: c * adversarial + squared L2 (L1 via proximal operator).

    The constant c balances misclassification vs distortion.
    Beta controls L1 vs L2 trade-off (sparsity vs smoothness).

    Parameters:
        adv_images (torch.Tensor): Current adversarial images
        original_images (torch.Tensor): Original clean images
        labels_onehot (torch.Tensor): One-hot encoded labels
        const (torch.Tensor): Trade-off constants c per example
        model (nn.Module): Target model
        beta (float): L1 weight in elastic-net distance
        confidence (float): Margin for misclassification
        targeted (bool): Whether this is a targeted attack

    Returns:
        tuple: (total_loss, adversarial_loss, distances)
    """# Get model predictionslogits=model(adv_images)# Compute adversarial lossadversarial_loss=compute_adversarial_loss(logits,labels_onehot,confidence,targeted)# Compute distancesl1_dist,l2_dist,elastic_dist=compute_distances(adv_images,original_images,beta)# Combine: c * adversarial_loss + L2_distance# Note: L1 is handled by FISTA's proximal operator, not in this gradienttotal_loss=const*adversarial_loss+l2_distreturntotal_loss,adversarial_loss,(l1_dist,l2_dist,elastic_dist)`
```

We begin by computing model outputs (logits) that drive the adversarial loss. A forward pass is the most expensive step in the pipeline and repeats hundreds of times during FISTA, so inference speed strongly affects overall runtime.



FISTA’s proximal operator enforces theL1L_1penalty, so the total loss omits it. Autograd then computes∇x′[c⋅f(x′,y)+∥x′−x∥22]\nabla_{x'}\,[c \cdot f(x', y) + \|x' - x\|_2^2]for the gradient step, while the shrinkage operation applies theL1L_1effect without needing its gradient.



This function returns three layers of information: the scalar total loss for backpropagation, the adversarial loss for monitoring success, and all three distance metrics for analysis. That richer signature supports detailed logging while still providing exactly the gradients we need.

Per-example`const`values let the binary search adapt to each example's difficulty. Easy cases converge to small constants and low distortion, while resistant cases require larger constants, accepting more distortion to achieve misclassification.

With loss functions defined, the next section implements the core FISTA building blocks: momentum computation for acceleration and soft thresholding for sparsity. These components combine with the loss functions to form the complete FISTA optimization step.
