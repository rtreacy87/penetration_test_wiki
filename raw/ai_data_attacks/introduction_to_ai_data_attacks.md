# AI Data Attacks

---

The previous section mapped the normal data pipeline, from collection through retraining. This section maps what goes wrong when an adversary targets that pipeline deliberately.

`AI data attacks`corrupt systems by going after the data the model learns from or the stored model artifact itself. That makes them fundamentally different from two other attack families covered in other topics.`Evasion attacks``manipulate inputs at inference time to fool a deployed model`. Privacy`attacks``extract sensitive information the model has memorized`.

The attacks here work earlier.

They`corrupt the data the model learns from or the format it is stored in`. By the time the model serves a prediction, the damage is already inside it.

![Diagram showing Attack Surface with objectives: Data Poisoning, Storage Tampering, Processing Manipulation, Model Poisoning, Deployment Injection, Retraining Exploitation, targeting AI Data Pipeline Stages: Data Collection, Storage, Data Processing, Analysis & Modeling, Deployment, Monitoring & Maintenance.](https://academy.hackthebox.com/storage/modules/302/pipeline_attacks.png)

Every pipeline stage has a corresponding attack type. The sections below walk through each one using the two concrete examples from the previous section (the e-commerce recommendation system and the healthcare diagnostic tool) so the attack descriptions stay grounded in systems we have already mapped.

![Diagram showing Adversary injecting malicious data into Data Collection Stage, leading to Data Poisoning, resulting in Corrupted Training Set and Compromised AI Model.](https://academy.hackthebox.com/storage/modules/302/data_collection_attacks.png)

Collection is exactly where`data poisoning`enters the pipeline. The attacker injects malicious data at the source, not by breaking in, but by simply using the same input channels the system already trusts. The injected data is crafted for a specific downstream effect, typically`label flipping`(changing what the model thinks a sample means) or`feature attacks`(perturbing the input values a sample carries).

Consider the e-commerce example. An attacker submits waves of fake positive reviews for a specific product.

Those reviews carry the correct format, arrive through the normal review submission API, and look entirely like legitimate user feedback. They are poisoned features and labels entering through a trusted channel. Some reviews might also contain specific keyword patterns, meaningless to a human reader, but designed to function as backdoor triggers.

If the model later receives an input containing those keywords, the backdoor activates. The model produces whatever output the attacker intended.

The healthcare example works somewhat differently. An attacker with access to the ingestion pathway alters`DICOM`metadata or manipulates clinical notes to mislabel diagnostic samples. These individual perturbations are small. A shifted pixel distribution in an image header. A swapped diagnosis code in a text note.

But if enough poisoned samples reach the training set, the CNN learns a version of the diagnostic task that inherently includes the attacker's distortions: wrong classifications, systemic biases, or triggered behaviors that only activate under specific input conditions.

![Diagram showing Adversary gaining unauthorized access to Storage System, leading to theft/tampering of datasets and models, and model replacement with Trojan, affecting AWS S3/Secure FS.](https://academy.hackthebox.com/storage/modules/302/stor_age_attacks.png)

Storage-layer attacks roughly split into two categories. The second is the one that matters heavily for AI-specific threats.

The first is straightforward: unauthorized access to the`AWS S3`data lake or the healthcare provider's secure storage lets an attacker steal or modify training datasets entirely. Post-collection tampering (such as changing labels or perturbing features in stored files) bypasses whatever validation existed at ingestion.

This is data poisoning through a different door.

The second targets model files. The`.pkl`recommendation model on`S3`and the`.pt`diagnostic model are high-value targets precisely because they carry executable trust.

An attacker with write access can obviously replace a legitimate model with one containing an embedded`trojan`, a model that behaves normally on clean inputs but produces attacker-chosen outputs on trigger inputs. But they can also go further and execute a`model steganography attack`: hiding arbitrary code directly inside the model file.

This works because deserialization mechanisms like`pickle.load()`execute embedded code when they load the file.

The effect is not wrong predictions. It is code execution. A compromised`.pkl`loaded by the`Flask`API server gives the attacker a solid foothold on the serving infrastructure, while a compromised`.pt`loaded by the clinical system gives access to the entire healthcare network.

![Diagram showing Adversary manipulating processing logic in Data Processing Stage, leading to corrupted processed data and indirect downstream model impact.](https://academy.hackthebox.com/storage/modules/302/processing_attacks.png)

Processing-stage attacks do not directly target data. They target the code that transforms data.

The distinction matters immensely. If an attacker compromises the cleaning, transformation, or feature engineering logic, they can corrupt data that was perfectly clean when it arrived. The raw data upstream still passes any audit. The corruption happens inside the pipeline's own processing step, which makes it significantly harder to detect.

Consider the e-commerce platform. Compromising the`Spark`job that runs sentiment analysis on reviews could easily flip sentiment labels: positive reviews get tagged negative, and negative reviews get tagged positive.

That is a textbook`label flipping`attack, but it did not require touching a single raw review. It required modifying a job configuration or injecting code into a processing script.

The healthcare analog works exactly the same way. Manipulating the`Python`scripts that standardize`DICOM`images or extract features from clinical notes could introduce subtle errors (shifted pixel normalizations, misextracted text features) that constitute`feature attacks`.

The raw images and notes remain correct in storage. The corruption exists only in the processed output the model trains on.

![Diagram showing Adversary influencing Analysis & Modeling Stage with corrupted data, leading to a compromised model with biases and incorrect patterns.](https://academy.hackthebox.com/storage/modules/302/modeling_attacks.png)

Modeling is where all upstream damage materializes.

The`AWS SageMaker`training job does not know that the`Parquet`files contain flipped labels. The`PyTorch`training loop does not know that some image features secretly carry backdoor triggers. Both blindly learn what the data teaches. Poisoned data produces a deeply poisoned model, one that has learned incorrect patterns, exhibits systematic biases, or contains hidden backdoor behavior that activates only when specific trigger inputs appear in production.

The attack surface at this stage is inherited, not introduced.

The adversary's work was already done during collection, at storage, or in processing. Modeling simply converts that earlier corruption into a trained artifact that will aggressively carry the manipulation forward into deployment.

![Diagram showing Adversary injecting malicious model file into Model Storage, leading to insecure loading in Production Environment, resulting in a compromised system.](https://academy.hackthebox.com/storage/modules/302/deployment_attacks.png)

The deployment stage introduces an entirely different attack vector: intercepting the model file between storage and the production environment.

If the loading mechanism lacks rigid integrity checks (no hash verification, no signature validation, or an insecure deserialization path) an attacker can swap in a malicious model file at the point of deployment rather than in storage.

The final outcome is the same`Trojan`or`model stenography`risk described previously, but the access requirement is fundamentally different. The attacker does not need direct write access to S3 or the model registry.

They only need access to the deployment path. A misconfigured CI/CD pipeline. An unauthenticated model-pull endpoint. A man-in-the-middle position securely wedged between storage and the serving pod.

![Diagram showing Adversary injecting malicious feedback into Retraining Pipeline, corrupting input and affecting Production Model, creating a retraining loop.](https://academy.hackthebox.com/storage/modules/302/monitoring_attacks.png)

Retraining is undoubtedly the most effective entry point for`online poisoning`, and the exact reason is structural: the pipeline is designed to implicitly trust new data.

Take the e-commerce platform's`Airflow`-managed retraining pipeline. The attacker does not need to awkwardly compromise storage or processing code. They just submit manipulated data through the same channels legitimate users do. Subtly altered clickstream data. Misleading feedback that reliably biases future labels. Repeated interactions perfectly engineered to pull model weights toward specific outcomes.

Each individual submission looks entirely normal. The poisoning steadily accumulates across long retraining cycles.

That gradual pace is what makes`online poisoning`incredibly hard to catch. Recommendation quality does not completely collapse overnight. It lazily drifts. Biases emerge slowly across many model versions, and the monitoring system essentially has to distinguish between three things that look incredibly similar: genuine distribution shift in user behavior, random noise, and active adversarial manipulation.

The system was purposely built to adapt to the first. It has literally no built-in mechanism to reject the third.

The overall impact of successful AI data attacks spans a wide range. At the mild end, subtly biased decisions and degraded prediction quality. At the severe end, absolute model compromise.

At the worst end, when the attack involves embedded trojans or code execution through deserialized model files, the breach dangerously extends beyond the model and directly into the broader infrastructure.

## Mapping Vulnerabilities to Security Frameworks

Two security frameworks provide structured language for the risks covered above.

The`OWASP Top 10 for LLM Applications`, introduced in the "[Introduction to Red Teaming AI](https://enterprise.hackthebox.com/content-library?s=Introduction%20to%20Red%20Teaming%20AI)" module, maps the data-level attacks directly.`Data poisoning`, meaning the corruption of data during collection, processing, training, or feedback, falls under`OWASP LLM03: Training Data Poisoning`. That single entry captures the majority of the attacks described in this section.

`OWASP LLM05: Supply Chain Vulnerabilities`covers the rest.

It comprehensively addresses compromised third-party data sources, tampered pre-trained model artifacts, and vulnerabilities in the various software components and platforms that make up the pipeline infrastructure.`LLM05`is much broader than data poisoning alone. It actively extends into dependency management, third-party model provenance, and robust infrastructure integrity.

Robust protection fundamentally requires secure system design principles that go beyond any single checklist entry: absolute access control, authentication, and authorization across every pipeline stage.

[Google's Secure AI Framework](https://saif.google/)(SAIF) covers the same ground from a lifecycle perspective rather than a simple vulnerability list.

![Flowchart showing Model Creation process: Data Sources to Data Filtering & Processing, Data Storage Infrastructure, Evaluation, Training & Tuning, and Model Storage Infrastructure.](https://academy.hackthebox.com/storage/modules/302/saif_data.png)

SAIF organizes its key principles around proper`Secure Design`, firmly protecting`Data`components, and adequately verifying the`Secure Supply Chain`.

Preventing data poisoning maps cleanly to securing the data supply chain, primarily by implementing rigid`Security Testing`during model development. This is critical for data actively entering through retraining loops, where the trust boundary is generally the weakest.

Model artifact integrity and code injection prevention heavily fall under SAIF's`Secure Deployment`practices. Successfully detecting data or actual model manipulation hidden inside running retraining loops elegantly maps to`Secure Monitoring & Response`, heavily treating cybersecurity as an ongoing continuous process, rather than a one-time deployment gate.

That last point strongly matters for this topic.

The highly specific attacks covered here, especially online poisoning, are structurally designed to operate silently within completely normal system behavior. Spotting them requires relentless monitoring geared directly toward identifying statistical anomalies across massive data distributions over time. Basic perimeter security at deployment isn't remotely enough.
