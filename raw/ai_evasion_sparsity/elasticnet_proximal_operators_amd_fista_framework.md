# Proximal Operators and FISTA Framework

The previous section configured our environment and trained a target model. Now we develop the mathematical machinery needed to optimize ElasticNet's mixed-norm objective. The question is far from trivial.



FGSM had it easy. Closed-form solution. Sign operations. Done.
DeepFool repeatedly solved linear approximations using closed-form
projection formulas. ElasticNet’s mixed objective admits no such simple
solution. TheL1L_1term creates a fundamental obstacle.







Why isL1L_1optimization so difficult? The absolute value function|x||x|has a sharp corner at zero where the derivative is undefined. Standard
gradient descent relies on smooth, continuous derivatives to determine
update directions. At exactly the points where sparsity occurs (values
becoming zero), the gradient doesn’t exist. Attempting naive gradient
descent on∥δ∥1\|\delta\|_1produces frustrating results: values hover near zero, getting smaller
and smaller, but rarely reach zero exactly. You get pseudo-sparsity
(many small values) instead of true sparsity (many exact zeros).



The`Fast Iterative Shrinkage-Thresholding Algorithm`(FISTA) decomposes the problem into smooth and non-smooth components, handling each with appropriate techniques. Proximal operators replace gradients for non-smooth terms, while Nesterov momentum accelerates convergence. Together, these tools enable efficient optimization of objectives that would defeat simpler methods.

## The Proximal Operator Framework

How do we generalize projection to handle non-smooth penalty functions? Proximal operators provide the answer. To understand this generalization, consider first the simpler case of constrained optimization that appeared in FGSM and DeepFool.



FGSM can be viewed as solving the linearized objective under anL∞L_\inftyconstraint: maximize∇J(x)⋅δ\nabla J(x) \cdot \deltasubject to∥δ∥∞≤ϵ|\delta|_\infty \leq \epsilon.
The solution isδ=ϵ⋅sign⁡(∇J(x))\delta = \epsilon \cdot \operatorname{sign}(\nabla J(x)).
DeepFool’s linear approximation similarly projects onto hyperplanes
representing decision boundaries, using the geometric formula for
distance from a point to a plane.







Proximal operators extend this projection idea to handle penalty
functions rather than hard constraints. Instead of asking "what’s the
closest point in setCC?",
we ask "what point minimizes the sum of distance-from-here plus some
penalty function?" Formally, for functionhhat pointzzwith step sizeλ\lambda:





proxλh(z)=arg⁡min⁡x{12∥x−z∥22+λh(x)}\text{prox}_{\lambda h}(z) = \arg\min_x \left\{ \frac{1}{2}\|x - z\|_2^2 + \lambda h(x) \right\}





What do these terms accomplish? The first term12∥x−z∥22\frac{1}{2}\|x - z\|_2^2measures squared distance from the input pointzz,
encouraging solutions close tozz.
The second termλh(x)\lambda h(x)applies the penalty function, encouraging solutions with desirable
properties according tohh.
The minimizer balances these competing objectives, finding a point that
is both close tozzand has small penalty underhh.





Different penalty functions produce different proximal operators.
Whenhhis an indicator function (infinite outside a set, zero inside), we
recover ordinary projection. Whenhhis a differentiable function like∥x∥22\|x\|_2^2,
we get a closed form involving gradient steps. Whenhhis non-smooth like∥x∥1\|x\|_1,
the proximal operator provides a well-defined operation that replaces
the problematic gradient.





For theL1L_1norm, the proximal operator has a particularly elegant form:
element-wise soft thresholding. This operation treats each coordinate
independently, applying a simple threshold rule that zeros out small
values and shrinks large ones. The threshold levelλ\lambdacontrols how aggressively sparsity is enforced: largerλ\lambdacreates sparser solutions by zeroing out more coordinates.



## Soft Thresholding: The L1 Proximal Operator

What is the proximal operator for



h(x)=∥x∥1h(x) = |x|*1?
The answer is elegant: soft thresholding. The operator𝒮λ\mathcal{S}*\lambda
acts element-wise according to this rule:







𝒮λ(z)i={zi−λifzi>λ0if|zi|≤λzi+λifzi<−λ\mathcal{S}_\lambda(z)_i = \begin{cases}z_i - \lambda & \text{if } z_i > \lambda \\\0 & \text{if } |z_i| \leq \lambda \\\z_i + \lambda & \text{if } z_i < -\lambda\end{cases}





To see the operator in action, setλ=0.1\lambda=0.1.
Forzi=0.12z_i=0.12,
soft thresholding returns𝒮0.1(z)i=0.02\mathcal{S}_{0.1}(z)_i = 0.02;
forzi=0.08z_i=0.08,
it returns00;
forzi=−0.25z_i=-0.25,
it returns−0.15-0.15.
In vector form, forz=[0.12,0.08,−0.25]z=[0.12,\,0.08,\,-0.25]we get𝒮0.1(z)=[0.02,0.0,−0.15]\mathcal{S}_{0.1}(z)=[0.02,\,0.0,\,-0.15].
This simple check illustrates the three-region rule: exact zeros near
the origin and shrinkage just outside that zone.





Three regions. Three actions. Values with magnitude belowλ\lambdaget zeroed out completely, creating exact sparsity. Positive values
aboveλ\lambdaare reduced byλ\lambda,
shrinking toward zero from above. Negative values below−λ-\lambdaare increased byλ\lambda,
shrinking toward zero from below. The name "soft thresholding"
distinguishes this from hard thresholding, which would zero out values
belowλ\lambdabut leave others unchanged (imagine cutting with scissors for hard
thresholding versus gradually squeezing for soft).





Why does this operation minimize the proximal objective? Consider a
single coordinate of the optimization problem. We want to findxix_ithat minimizes12(xi−zi)2+λ|xi|\frac{1}{2}(x_i - z_i)^2 + \lambda |x_i|.
The first term is a quadratic pullingxix_itowardziz_i.
The second term is an absolute value penalty pullingxix_itoward zero. These competing forces balance at the soft threshold. To
see this mathematically, split the problem into three cases based onziz_i.





Forzi>λz_i > \lambda,
assume the minimum occurs at positivexi>0x_i > 0,
making|xi|=xi|x_i| = x_i.
Taking the derivative and setting to zero:xi−zi+λ=0x_i - z_i + \lambda = 0,
givingxi=zi−λx_i = z_i - \lambda.
This confirms the first case of soft thresholding. Forzi<−λz_i < -\lambda,
assuming negativexi<0x_i < 0makes|xi|=−xi|x_i| = -x_i.
The derivative equation becomesxi−zi−λ=0x_i - z_i - \lambda = 0,
givingxi=zi+λx_i = z_i + \lambda,
confirming the third case. For|zi|≤λ|z_i| \leq \lambda,
we can verify thatxi=0x_i = 0achieves lower objective value than any non-zeroxix_i,
confirming the middle case.





Whenziz_iis small (magnitude less thanλ\lambda),
theL1L_1penalty’s pull toward zero dominates the quadratic’s pull towardziz_i.
The minimum occurs at exactly zero. Whenziz_iis large, the quadratic term dominates nearziz_i,
but theL1L_1penalty still exerts constant downward pressure, causing the minimum to
occur atziz_ioffset byλ\lambdatoward zero. This analysis confirms that soft thresholding solves the
one-dimensional proximal problem exactly.





The full proximal operator applies this element-wise operation across
all coordinates. Because theL1L_1norm decomposes as a sum of absolute values, and the squaredL2L_2norm similarly decomposes as a sum of squares, the multi-dimensional
proximal problem separates into independent one-dimensional problems.
Each coordinate can be thresholded independently without considering the
others, making the operation computationally efficient and easily
parallelizable.





Contrast this with theL2L_2proximal operator, which must consider all coordinates jointly. The
proximal operator forh(x)=∥x∥2h(x) = \|x\|_2producesproxλh(z)=max⁡(0,1−λ/∥z∥2)⋅z\text{prox}_{\lambda h}(z) = \max(0, 1 - \lambda/\|z\|_2) \cdot z,
a scaling operation that treats the entire vector as a unit. This
operator shrinks vectors uniformly rather than creating sparsity by
zeroing individual coordinates.



## From Proximal Operators to FISTA

Proximal gradient methods solve problems of the form



min⁡x{f(x)+h(x)}\min_x { f(x) + h(x) }whereffis smooth buthhis not. The strategy decomposes each iteration into two steps. First,
take a standard gradient step on the smooth functionff,
moving from current pointx(k)x^{(k)}in the direction−∇f(x(k))-\nabla f(x^{(k)})with step sizeη\eta.
Second, apply the proximal operator ofhhto handle the non-smooth term.





The basic proximal gradient update is:



x(k+1)=proxηh(x(k)−η∇f(x(k)))x^{(k+1)} = \text{prox}_{\eta h}\left( x^{(k)} - \eta \nabla f(x^{(k)}) \right)







This iteration provably converges to a minimizer under standard
assumptions onffandhh.
The convergence rate isO(1/k)O(1/k)wherekkis the iteration count, meaning the error decreases inversely with
iterations. While this beats naive subgradient methods, it remains
relatively slow for problems requiring high precision.





FISTA accelerates this basic scheme using Nesterov momentum. Instead
of computing gradients at the current pointx(k)x^{(k)},
FISTA computes them at an extrapolated pointy(k)y^{(k)}that anticipates where the optimization is heading. This look-ahead
allows the algorithm to better exploit consistent directions in the
objective landscape, achieving faster convergence.



The FISTA update sequence is:



x(k+1)=proxηh(y(k)−η∇f(y(k)))x^{(k+1)} = \text{prox}_{\eta h}\left( y^{(k)} - \eta \nabla f(y^{(k)}) \right)







tk+1=1+1+4tk22t_{k+1} = \frac{1 + \sqrt{1 + 4t_k^2}}{2}





y(k+1)=x(k+1)+tk−1tk+1(x(k+1)−x(k))y^{(k+1)} = x^{(k+1)} + \frac{t_k - 1}{t_{k+1}}(x^{(k+1)} - x^{(k)})





The variabletkt_kcontrols the momentum coefficient. Starting fromt0=1t_0 = 1,
this sequence grows with iterations, providing progressively stronger
acceleration. The specific formula fortkt_kis carefully chosen to ensure the convergence guarantee: FISTA achievesO(1/k2)O(1/k^2)convergence rate, a quadratic improvement over standard proximal
gradient descent.





In practice, the momentum update can be simplified. The papers
introducing FISTA showed thattk≈k/2t_k \approx k/2for largekk,
leading to momentum coefficient(tk−1)/tk+1≈(k−2)/(k+1)(t_k - 1)/t_{k+1} \approx (k-2)/(k+1).
For implementation simplicity, many practitioners use the approximationk/(k+3)k/(k+3),
which provides similar acceleration behavior while being slightly more
conservative. This approximation maintains fast convergence while
improving numerical stability.



## Applying FISTA to ElasticNet

Now we apply FISTA to ElasticNet's specific attack objective, decomposing it into components that optimization can handle efficiently.



ElasticNet pairs adversarial loss, which drives misclassification,
with mixed-norm distance penalties that control perturbation size and
sparsity. We use the Carlini & Wagner margin formulation, measuring
how much the true class logitZy(x′)Z_y(x')exceeds the strongest competitor. The distance term combines squaredL2L_2for smoothness withL1L_1for sparsity, weighted byβ\beta.
A trade-off constantccbalances these objectives.







For ElasticNet, decomposing into smooth and non-smooth parts follows
naturally from this structure. Define the smooth partffas the adversarial loss plus theL2L_2distance:





f(x′)=c⋅max⁡(Zy(x′)−max⁡j≠yZj(x′)+κ,0)+∥x′−x∥22f(x') = c \cdot \max\Big(Z_y(x') - \max_{j \neq y} Z_j(x') + \kappa, 0\Big) + \|x' - x\|_2^2





Define the non-smooth parthhas theL1L_1penalty:





h(x′)=β∥x′−x∥1h(x') = \beta \|x' - x\|_1





Gradients forffcombine backpropagation through the neural network (for the adversarial
loss) with a simple linear gradient (for theL2L_2term). Both components are differentiable almost everywhere, with the
adversarial loss’s hinge being handled by standard subgradient
conventions. TheL2L_2gradient is particularly simple:∇x′∥x′−x∥22=2(x′−x)\nabla_{x'} \|x' - x\|_2^2 = 2(x' - x).





Soft thresholding is the proximal operator forhh,
applied to the perturbation. Withδ=x′−x\delta = x' - xfor the perturbation, we have:





proxηβ∥⋅∥1(z)=x+𝒮ηβ(z−x)\text{prox}_{\eta \beta \|\cdot\|_1}(z) = x + \mathcal{S}_{\eta \beta}(z - x)



This formulation makes explicit that sparsity is enforced on the perturbation itself, not the adversarial image. We want sparse perturbations (many pixels unchanged), not sparse images (many pixels zero).



Each FISTA iteration performs these steps in sequence. Compute the
gradient of smooth terms at the extrapolated pointy(k)y^{(k)}.
Take a gradient step with learning rateη\eta.
Apply soft thresholding with thresholdηβ\eta \betato enforceL1L_1sparsity. Update the momentum pointy(k+1)y^{(k+1)}using the simplified formulak/(k+3)k/(k+3).
This sequence repeats for hundreds of iterations until convergence, withη\etacontrolling the gradient step size.







To see how the proximal step and momentum interact, consider a
two-dimensional example. After a gradient step, suppose the candidate
perturbation relative to the original isδ=[0.12,−0.07]\delta=[0.12,\,-0.07]and setβ=0.1\beta=0.1.
Soft thresholding gives𝒮0.1(δ)=[0.02,0.0]\mathcal{S}_{0.1}(\delta)=[0.02,\,0.0].
The first coordinate survives with shrinkage; the second drops to zero,
which enforces sparsity on small changes. Withk=10k=10,
the momentum coefficientk/(k+3)=10/13≈0.769k/(k+3)=10/13\approx 0.769extrapolates the next evaluation point along the update direction, using
recent progress to look ahead and speed convergence when updates keep
pointing the same way.



## Why FISTA for ElasticNet?

ElasticNet's iterative optimization differs fundamentally from FGSM's single-step approach and DeepFool's geometric linearization. The specific optimization mechanics make this difference clear.



FGSM requires no iterative optimization. One gradient evaluation
yields the attack direction through a sign operation. Total iterations:
one. DeepFool iterates through geometric projections, solving 5-20
closed-form linear problems without gradient descent or line search.
FISTA performs true iterative optimization through gradient descent with
proximal operators, requiring 100-1000 iterations where each step
computes gradients via full backpropagation, applies soft thresholding
forL1L_1sparsity, and updates momentum for acceleration.







Why this complexity? ElasticNet’s mixedL1/L2L_1/L_2objective admits no closed-form solution. TheL1L_1term creates non-smooth corners where standard gradients fail. The
misclassification constraint couples perturbations to the neural
network’s decision boundary in ways that resist geometric
simplification. FISTA handles this complexity by decomposing the problem
into manageable pieces: gradient descent for smooth terms, proximal
operators for non-smooth terms, and momentum for acceleration.



The following sections build toward the complete FISTA implementation. We begin by defining the loss functions that provide gradients: distance metrics quantify perturbation magnitude, adversarial loss drives misclassification, and the total loss combines these components. With loss functions established, we then implement the FISTA building blocks: momentum computation, soft thresholding, and the complete iteration step. Finally, we integrate these pieces with binary search to produce minimal adversarial perturbations.
