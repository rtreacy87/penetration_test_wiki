# Distance Metrics



To balance sparsity against smoothness, ElasticNet needs three
distance computations:L1L_1for counting total change, squaredL2L_2for measuring energy, and their weighted combination for the actual
optimization objective. Computing these efficiently on batched tensors
while preserving per-example metrics requires careful dimension
handling.



## Measuring Perturbation Magnitude



TheL1L_1distance tells us the total magnitude of change summed across all
pixels. For a28×2828 \times 28MNIST image with 784 pixels, this sum ranges from 0 (no change) to
potentially hundreds (many pixels changed substantially). If 100 pixels
each change by 0.15 units, theL1L_1distance is100×0.15=15.0100 \times 0.15 = 15.0.
If instead 50 pixels each change by 0.30 units, theL1L_1distance is still50×0.30=15.050 \times 0.30 = 15.0.
UnlikeL0L_0which counts changed pixels (100 versus 50),L1L_1weights each pixel by how much it changed.





TheL2L_2distance measures the squared straight-line distance in 784-dimensional
pixel space. Squaring individual perturbations before summing creates a
strong penalty for large individual changes. Consider the same two
perturbations: 100 pixels at 0.15 each yieldsL2=100×0.152=2.25L_2 = 100 \times 0.15^2 = 2.25,
while 50 pixels at 0.30 each yieldsL2=50×0.302=4.50L_2 = 50 \times 0.30^2 = 4.50.
The second perturbation has twice theL2L_2distance despite identicalL1L_1distance, becauseL2L_2penalizes concentrated changes more heavily than distributed
changes.





The elastic-net distance∥δ∥22+β∥δ∥1\|\delta\|_2^2 + \beta \|\delta\|_1combines both norms with parameterβ\betacontrolling their relative weight. Whenβ=0\beta = 0,
we recover pureL2L_2distance. Asβ\betaincreases, theL1L_1component gains influence, biasing optimization toward sparser
solutions. This mixed distance lacks the clean geometric interpretation
of pure norms, but it captures the sparsity-smoothness balance that pure
norms cannot.



## Understanding Norm Trade-offs



Different norms encode different notions of perturbation quality.
PureL2L_2optimization produces smooth, distributed perturbations where many
pixels change by small amounts. This distributes the perturbation budget
evenly, avoiding concentrated modifications that might be visually
obvious. However,L2L_2alone provides no sparsity guarantee. AnL2L_2-optimal
perturbation might modify every single pixel by a tiny amount, achieving
low Euclidean distance while creating perceptually noticeable global
shifts in brightness or texture.





PureL1L_1optimization encourages sparsity through its non-differentiable kink at
zero. The absolute value function has no unique derivative at the
origin, creating a "barrier" that optimization must overcome to activate
a pixel. This barrier preferentially keeps small perturbations at
exactly zero, concentrating the perturbation budget into fewer pixels
with larger individual changes. For a fixedL1L_1budget, modifying 50 pixels at 0.2 each gives the sameL1L_1distance as modifying 100 pixels at 0.1 each, but the sparse solution
(50 pixels) often proves more effective at fooling classifiers because
concentrated changes can cross local decision boundaries more
decisively.





The elastic-net combination provides a middle ground. TheL2L_2term prevents any single pixel from dominating the perturbation (since
squaring penalizes large values quadratically), while theL1L_1term encourages many pixels to remain at exactly zero. Tuningβ\betacontrols this balance. Smallβ\beta(like 0.01) produces nearly-smooth perturbations with mild sparsity,
suitable when imperceptibility matters more than pixel count. Largeβ\beta(like 1.0) produces aggressive sparsity, potentially concentrating
perturbations into visually obvious clusters but minimizing the number
of modified pixels.



Code:python```
`defcompute_distances(adv_images,original_images,beta):"""
    Compute L1, L2, and elastic-net distances.

    Returns all three distance metrics used in optimization
    and decision-making.

    Parameters:
        adv_images (torch.Tensor): Adversarial images (batch_size, C, H, W)
        original_images (torch.Tensor): Original images (batch_size, C, H, W)
        beta (float): Weight for L1 in elastic-net distance

    Returns:
        tuple: (l1_dist, l2_dist, elastic_dist) each shape (batch_size,)
    """l1_dist=torch.sum(torch.abs(adv_images-original_images),dim=(1,2,3))l2_dist=torch.sum((adv_images-original_images)**2,dim=(1,2,3))elastic_dist=l2_dist+beta*l1_distreturnl1_dist,l2_dist,elastic_dist`
```



The`dim=(1, 2, 3)`parameter collapses channel, height,
and width while preserving the batch dimension. For a batch of 20 images
with shape (20, 1, 28, 28), this produces 20 scalar distances - one per
image. Why preserve the batch dimension? Binary search adapts trade-off
constants individually: an easy example near the decision boundary might
usec=0.01c=0.01while a resistant example deep in the correct class region needsc=10c=10.
Without per-example distances, we’d be forced to use the same constant
for all images, over-perturbing easy cases or under-perturbing hard
ones.





Notice we’re using squaredL2L_2distance∥x′−x∥22\|x' - x\|_2^2for the same computational reasons discussed in the FISTA framework: the
gradient2(x′−x)2(x' - x)is simpler than the normalized form from the trueL2L_2norm. This maintains strict convexity and produces the same optimal
perturbations. The elastic distance combines the squaredL2L_2component with theL1L_1term, using the sameβ\betaweighting that appears in our proximal operator.



Note what this function does not do: it doesn't apply any constraints, perform any thresholding, or make any decisions. It simply measures. Other functions will use these distances to make decisions about attack success or to compute loss gradients, but the distance computation itself remains a pure measurement operation.

With distance metrics defined, the next section implements the adversarial loss functions that drive misclassification and combines them with these distance terms to form the complete optimization objective.
