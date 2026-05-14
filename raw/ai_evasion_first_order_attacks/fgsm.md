# Understanding Norms

When we talk about adversarial attacks on machine learning models, one of the most important questions is: "How much did we change the input?" This might seem simple - just look at the image and see if it looks different, but computers need precise mathematical ways to measure these changes, and that's where`norms`come in.

Think of a norm as a ruler for measuring changes. Just like you might measure distance in miles or kilometers, we use different norms to measure how much an adversarial attack has modified an input. The fascinating part is that different "rulers" (norms) measure changes in completely different ways, leading to very different types of attacks.

## What Exactly Is a Norm?

![City grid metaphor comparing L0 counting turns, L1 Manhattan distance, L2 diagonal distance, and L∞ longest single leg between start and goal.](https://academy.hackthebox.com/storage/modules/319/introduction_norm_city_metaphor.png)

Let's start with something familiar. Imagine you're in New York City and want to travel from Times Square to Central Park. How far is it? The answer depends on how you measure:

- If you count how many intersections you pass through (regardless of distance), that's one measure
- If you have to walk along the city streets (following the grid), that's a different distance
- If you can fly in a straight line (like a bird), that's another distance
- If you only care about the longest single stretch you have to travel, that's yet another way to measure
These different ways of measuring distance are exactly what norms do for adversarial perturbations. A`norm`is simply a mathematical tool that assigns a "size" or "length" to changes we make to an input.

## The Three Rules Every Norm Must Follow

For something to qualify as a proper norm, it must follow three common-sense rules:

1. `Zero means zero`: The only thing with zero length is... nothing. If your measurement says something has zero size, it better actually be zero. This prevents our "ruler" from lying to us.
1. `Doubling means doubling`: If you make a change twice as big, the measurement should also be twice as big. This keeps our measurements consistent and predictable.
1. `Shortcuts don't exist`: The direct path between two points is never longer than going the roundabout way. In math terms, this is called the "triangle inequality" - imagine a triangle where one side can't be longer than the other two sides combined.
These rules ensure our measurement system makes sense and behaves predictably.

## The p-Norm Family: Different Rulers for Different Jobs

The most common norms we use are called`p-norms`, and they're like a family of related measuring tools. The "p" is just a number that determines which family member we're using. Here's the formula:



∥x∥p=(∑i=1n|xi|p)1/p\|x\|_p = \left(\sum_{i=1}^{n} |x_i|^p\right)^{1/p}



This formula is just saying: "Take each change, raise it to the power p, add them all up, then take the p-th root." Different values of p give us different measuring tools:



Whenp=1p = 1,
we get theL1L_1norm (like walking in Manhattan). Whenp=2p = 2,
we get theL2L_2norm (like flying in a straight line). Whenp=∞p = \infty,
we get theL∞L_\inftynorm (just look at the biggest change).





As p increases, the norm increasingly focuses on larger values. Withp=1p = 1,
all changes contribute equally to the sum. Withp=2p = 2,
bigger changes matter more because of the squaring. And as p approaches
infinity, only the single largest change matters at all.





But wait, what aboutL0L_0?
Here’s where things get interesting. TheL0L_0"norm" is actually an imposter in the p-norm family. It doesn’t follow
the formula above at all. You can’t setp=0p = 0in that equation and get something that counts non-zero elements.
Instead,L0L_0has its own completely different definition:





∥x∥0=∑i𝟙[xi≠0]\|x\|_0 = \sum_{i} \mathbb{1}[x_i \neq 0]





This just counts how many elements are non-zero, regardless of their
magnitude. A pixel changed by 0.001 counts the same as one changed by
255. Despite not being a true p-norm (or even a proper norm, since it
violates the scaling rule), we useL0L_0constantly in adversarial ML because it captures something the p-norms
can’t: pure sparsity. It answers the question "how many things did we
touch?" rather than "how much did we change them?"



### The Shape of Constraints

Here's where it gets interesting. When we limit perturbations to a maximum norm value, we're essentially drawing a boundary that says "you can change the input, but only within this shape."

![Four constraint diagrams showing L0 selecting sparse points, L1 forming a diamond, L2 forming a circle, and L∞ forming a square as the perturbation budget grows.](https://academy.hackthebox.com/storage/modules/319/introduction_norm_shapes.png)

Imagine you're standing at a point and can move a fixed "distance" in any direction:



With theL0L_0norm, you can move in directions that change at mostkkcoordinates (you can changekkpixels to any value within the valid range). In continuous spaces, this
feasible set is a union of axis‑aligned subspaces and is unbounded
unless magnitudes are also bounded. With theL1L_1norm, you can reach anywhere within a diamond shape. With theL2L_2norm, you can reach anywhere within a circle. With theL∞L_\inftynorm, you can reach anywhere within a square.





These shapes matter because they determine which directions are
"cheaper" to move in. WithL0L_0,
you get complete freedom in how much you change the selected pixels, but
strict limits on how many you can touch; in practice, we often pair anL0L_0limit with a per‑pixel magnitude bound (e.g., the valid image range) to
make the set well‑behaved. WithL1L_1,
moving diagonally costs more than moving along the axes. WithL∞L_\infty,
you can move equally far in any direction as long as no single
coordinate changes too much.



## How Different Norms Relate to Each Other

Different norms aren't completely independent - they're connected by mathematical relationships. Think of it like converting between miles and kilometers. For any set of changes in an n-dimensional space:



∥x∥∞≤∥x∥2≤∥x∥1≤n⋅∥x∥2≤n⋅∥x∥∞\|x\|_\infty \leq \|x\|_2 \leq \|x\|_1 \leq \sqrt{n} \cdot \|x\|_2 \leq n \cdot \|x\|_\infty





And for theL0L_0norm, which counts non-zero elements:





∥x∥0≤nand∥x∥2≤∥x∥0⋅∥x∥∞\|x\|_0 \leq n \quad \text{and} \quad \|x\|_2 \leq \sqrt{\|x\|_0} \cdot \|x\|_\infty



This means if you limit changes using one of the`p‑norms`, you automatically get some limits on the others too. It's like saying "if you can only walk 10 blocks in Manhattan distance, there's also a limit to how far you could have flown in a straight line."



ForL0L_0,
there’s an extra wrinkle:`counting how many coordinates you change does not, by itself, bound how large those changes can be`.
To translate anL0L_0limit into bounds onL1L_1orL2L_2,
you also need a per‑coordinate magnitude bound (e.g., anL∞L_\inftycap or the valid pixel range). The inequality above,|x|2≤|x|*0,|x|*∞|x|_2 \le \sqrt{|x|*0},|x|*\infty,
is exactly this combination: sparsity`plus`a size
limit.



## The L0 Norm: Counting What Changed

![L0 norm triptych showing the clean H, sparse signed perturbation markers, and the adversarial H altered at only a few pixels.](https://academy.hackthebox.com/storage/modules/319/introduction_norm_l0.png)



Now let’s dive into the specific norms used in adversarial attacks.
TheL0L_0norm is the simplest conceptually - it just counts how many pixels (or
features) you changed. Period. It doesn’t care if you changed a pixel by
a tiny amount or completely flipped it from black to white. Changed is
changed.





∥δ∥0=∑i𝟙[δi≠0]\|\delta\|_0 = \sum_{i} \mathbb{1}[\delta_i \neq 0]





This formula uses an "indicator function" (the𝟙\mathbb{1}symbol) that returns 1 if something changed and 0 if it didn’t. Then it
adds up all the 1s to count the total changes.





Imagine you’re editing a photo and you can only use the paintbrush
tool on 10 pixels. You could make those 10 pixels any color you want,
but you can only touch 10 of them. That’s anL0L_0constraint.



Advantages:

- Very interpretable - you know exactly which parts of the input matter
- Can make large changes to a few pixels while leaving the rest untouched
Disadvantages:

- Those few changed pixels might stand out like sore thumbs
- Mathematically difficult to optimize (it's not smooth or convex)
Fun fact: The L0 "norm" technically isn't a true norm because it violates rule #2 (doubling the changes doesn't double the count). But everyone calls it a norm anyway because it's useful.

## The L1 Norm: Total Change Budget

![L1 norm triptych showing the clean H, scattered positive and negative perturbation pixels, and the adversarial H dotted with salt-and-pepper noise.](https://academy.hackthebox.com/storage/modules/319/introduction_norm_l1.png)



TheL1L_1norm adds up the absolute value of all changes. It’s like having a
budget for how much total change you can make, and you can spread it
around however you want.





∥δ∥1=∑i|δi|\|\delta\|_1 = \sum_{i} |\delta_i|



Think of it like having 100 pixels to distribute among modifications. You could:

- Make 100 pixels each 1% different
- Make 10 pixels each 10% different
- Make 1 pixel 100% different
- Any combination that adds up to 100%


TheL1L_1norm tends to create`sparse`perturbations, it naturally
concentrates the budget on a`subset`of coordinates rather
than spreading tiny changes everywhere. This happens because of its
diamond-shaped constraint boundary, which has "corners" that favor
solutions with many zeros; the selected coordinates may change more,
while many others stay exactly the same.



Advantages:

- Creates a nice balance between L0 and L2 behaviors
- Mathematically easier to work with than L0 (it's convex)
- Perturbations often look like scattered specks of noise
Disadvantages:

- Not as smooth to optimize as L2
- Can create visible "salt and pepper" noise patterns
## The L2 Norm: Smooth, Even Changes

![L2 norm triptych showing the clean H, a dense perturbation heatmap, and the softly blurred adversarial H with evenly distributed changes.](https://academy.hackthebox.com/storage/modules/319/introduction_norm_l2.png)



TheL2L_2norm is the famous Euclidean distance - straight-line distance in space.
It measures perturbations by taking the square root of the sum of
squared changes:





∥δ∥2=∑iδi2\|\delta\|_2 = \sqrt{\sum_{i} \delta_i^2}



This is the same distance formula you learned in geometry class! The squaring operation makes this norm heavily penalize large individual changes. If you try to change one pixel by a lot, the square of that change becomes huge. So L2 perturbations naturally spread changes evenly across all pixels.

Imagine you're adding a thin layer of fog to an image. The fog affects everything equally, making the whole image slightly different rather than noticeably changing specific parts.

Advantages:

- Creates the smoothest, most imperceptible perturbations
- Mathematically well‑behaved (convex and smooth away from 0). In practice, many methods use the`squared`L2 norm, which is differentiable everywhere and especially convenient for optimization.
- Well-understood optimization properties
Disadvantages:

- Changes everything at least a little bit
- Less interpretable - harder to see which features matter most
- May create a visible "haze" over the entire image
## The L∞ Norm: Maximum Change Limit

![L∞ norm triptych showing the clean H, a bounded perturbation heatmap, and the adversarial H with uniform clipped noise across the strokes.](https://academy.hackthebox.com/storage/modules/319/introduction_norm_linf.png)



TheL∞L_\inftynorm (L-infinity) only cares about the biggest single change you made.
It’s like a speed limit - you can go any speed you want as long as you
never exceed the limit.





∥δ∥∞=max⁡i|δi|\|\delta\|_\infty = \max_i |\delta_i|





With anL∞L_\inftyconstraint of 0.1, every pixel can change by up to 10%, but none can
change by more than that. This creates perturbations where many pixels
might be changed by similar amounts, up to the maximum allowed.



Advantages:

- Very intuitive constraint - "no pixel changes by more than X"
- Common in adversarial training defenses
- Creates uniform-looking perturbations
Disadvantages:

- Might change many pixels unnecessarily
- The hard maximum can be restrictive for optimization
## Computing with Different Norms

![Gradient flow comparison panels contrasting L0 jumpy steps, L1 axis-aligned paths, L2 smooth diagonal flow, and L∞ projections onto a square boundary.](https://academy.hackthebox.com/storage/modules/319/introduction_norm_gradient_flow.png)

From a practical standpoint, different norms have different computational properties:



The L0 norm is discontinuous and non-convex, often requiring greedy
algorithms or combinatorial search. It’s the "expert mode" of norm
constraints. The L1 norm has a "kink" at zero (the absolute value
function isn’t smooth there) and requires special optimization
techniques like soft-thresholding or proximal gradient methods. The L2
norm is smooth and differentiable everywhere, with gradients flowing
nicely, making optimization straightforward. It’s the "vanilla ice
cream" of optimization - reliable and well-behaved. The L∞ norm is
efficiently computed but has sparse gradients (only the maximum element
has a non-zero gradient) and is often handled with projected gradient
descent.



## The Bottom Line

![Row of H digits comparing visual effects of L0 sparse flips, L1 speckles, L2 haze, and L∞ evenly clipped perturbations.](https://academy.hackthebox.com/storage/modules/319/introduction_norm_side_by_side_row.png)



Norms are the primary language we use to measure and constrain
adversarial perturbations. Each norm -L0L_0,L1L_1,L2L_2,
andL∞L_\infty- offers different trade-offs between imperceptibility, computational
efficiency, and attack effectiveness. Understanding these trade-offs is
crucial for both creating strong attacks and building robust
defenses.



As the field evolves, we're seeing more sophisticated perturbation metrics that better capture human perception. But the classical norms remain central to adversarial ML because they provide a precise, mathematical framework for reasoning about attacks and defenses.
