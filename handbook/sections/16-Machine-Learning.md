# Section 16 — Machine Learning

> **Purpose**: AWS offers a spectrum of ML services, from fully managed APIs that require no ML expertise, to fully customizable training environments for research scientists. A solutions architect needs to understand when to use managed AI services versus building custom models, and how to integrate ML into production architectures.
>
> **Official Documentation**: [SageMaker](https://docs.aws.amazon.com/sagemaker/) | [Rekognition](https://docs.aws.amazon.com/rekognition/) | [Comprehend](https://docs.aws.amazon.com/comprehend/) | [Bedrock](https://docs.aws.amazon.com/bedrock/)

---

## 1. The ML Service Spectrum

| Level of Control | Service | Use Case |
|-----------------|---------|----------|
| **Fully managed API** | Rekognition, Polly, Transcribe, Translate, Comprehend | Standard AI tasks (image labels, text-to-speech, sentiment) |
| **Managed platform** | SageMaker Canvas, Bedrock | Build models without code; foundation models via API |
| **Managed training** | SageMaker Training | Custom model training with managed infrastructure |
| **Full control** | SageMaker Notebooks, EC2 with GPUs | Research, custom architectures, maximum flexibility |

> **Architectural Principle**: Start with managed APIs. Only move to custom training if managed services cannot meet accuracy or latency requirements. Custom ML is expensive in both compute cost and engineering time.

---

## 2. Amazon SageMaker

### 2.1 SageMaker Components

| Component | Purpose |
|-----------|---------|
| **Notebooks** | Jupyter notebooks for exploration and development |
| **Training Jobs** | Distributed model training on managed GPU/CPU clusters |
| **Processing Jobs** | Data preprocessing, feature engineering, validation |
| **Endpoints** | Real-time model hosting with auto-scaling |
| **Batch Transform** | Offline inference on large datasets |
| **Pipelines** | ML workflow orchestration (data prep → training → evaluation → deployment) |
| **Model Registry** | Version and approval workflow for model artifacts |
| **Feature Store** | Centralized storage for ML features |

### 2.2 SageMaker Deployment Options

| Option | Latency | Cost Model | Use Case |
|--------|---------|-----------|----------|
| **Real-time Endpoint** | Milliseconds | Per instance-hour | Online predictions (recommendations, fraud detection) |
| **Batch Transform** | Minutes to hours | Per job duration | Large-scale offline scoring (churn prediction, marketing) |
| **Asynchronous Inference** | Minutes | Per instance-hour + queue storage | Large payload inference (video, large images) |
| **Serverless Inference** | Higher cold start | Per invocation | Sporadic traffic, variable load |

### 2.3 SageMaker Inference Optimization

| Technique | Effect |
|-----------|--------|
| **Multi-model endpoints** | Host hundreds of models on one endpoint (cost savings) |
| **Multi-container endpoints** | Different frameworks on one endpoint |
| **Inference Recommender** | Auto-select optimal instance type for your model |
| **Compilation (Neo)** | Optimize model for specific hardware (Inferentia, GPU, ARM) |
| **Elastic Inference** | Attach fractional GPU to CPU instances (deprecated; use Inferentia or GPU instances) |

---

## 3. Managed AI Services

### 3.1 Amazon Rekognition

Computer vision API:
- **Object detection**: Identify objects, scenes, activities
- **Facial analysis**: Detect faces, attributes, emotions
- **Face recognition**: Match faces against collections
- **Text in image**: OCR for signage, documents
- **Content moderation**: Detect inappropriate content
- **Custom labels**: Train Rekognition on your specific object categories

### 3.2 Amazon Comprehend

NLP API:
- **Sentiment analysis**: Positive, negative, neutral, mixed
- **Entity recognition**: People, organizations, locations, dates
- **Key phrase extraction**: Important terms
- **Language detection**: Identify language of text
- **PII detection**: Identify and redact personal information
- **Custom classification**: Train on your document categories

### 3.3 Amazon Bedrock

Foundation model platform:
- Access to FMs from Anthropic (Claude), AI21 Labs (Jurassic), Stability AI (Stable Diffusion), Amazon (Titan), and others
- **API-based**: No model management required
- **Fine-tuning**: Customize models with your data
- **Agents**: Build applications that can take actions based on FM reasoning
- **Knowledge Bases**: RAG (Retrieval Augmented Generation) with OpenSearch or vector databases

> **Bedrock vs SageMaker for LLMs**: Bedrock is for using pre-trained FMs via API. SageMaker is for training, fine-tuning, and deploying your own models. Most applications should start with Bedrock.

---

## 4. ML Infrastructure

### 4.1 AWS Trainium and Inferentia

AWS-designed ML chips for cost-efficient training and inference:

| Chip | Purpose | Instance Types | Cost vs GPU |
|------|---------|---------------|-------------|
| **Trainium** | Training | Trn1, Trn2 | Up to 50% lower than GPU instances |
| **Inferentia** | Inference | Inf1, Inf2 | Up to 70% lower cost per inference than GPU |

**Use case**: High-scale inference where latency requirements allow (Trainium/Inferentia may have higher latency than GPU for some models but dramatically lower cost). Requires model compilation with AWS Neuron SDK.

---

## Architectural Decision Challenges

* **Scenario:** You need to add object detection, facial analysis, or text recognition to an application.
  * **Design:** Amazon Rekognition. Because it provides pre-trained computer vision APIs without requiring ML expertise.

* **Scenario:** You need to extract sentiment, entities, and key phrases from text.
  * **Design:** Amazon Comprehend. Because it is a fully managed NLP service that provides pre-trained models via API.

* **Scenario:** You need to build, train, and deploy custom machine learning models.
  * **Design:** Amazon SageMaker. Because it provides a complete managed platform for the ML lifecycle, including notebooks, training jobs, and endpoints.

* **Scenario:** You want to integrate foundation models (LLMs) via API without managing infrastructure.
  * **Design:** Amazon Bedrock. Because it is a serverless platform providing access to leading foundation models.

* **Scenario:** You need to reduce the cost of large-scale ML training or high-throughput inference.
  * **Design:** AWS Trainium or Inferentia instances. Because these purpose-built AWS chips offer significantly lower costs than traditional GPUs.

---

## 5. Points to Remember

- **Start with managed AI services** (Rekognition, Comprehend, Bedrock) before building custom models.
- **SageMaker Pipelines provide CI/CD for ML** — version, test, and deploy models like application code.
- **Feature Store centralizes ML features** — prevents training/serving skew.
- **Multi-model endpoints reduce hosting costs** when serving many similar models.
- **Bedrock provides API access to foundation models** — no infrastructure management for LLM applications.
- **Trainium and Inferentia reduce ML costs** — compile models with Neuron SDK for optimal performance.
- **Real-time endpoints for online inference; Batch Transform for offline scoring** — choose based on latency requirements.

---

## 6. Further Reading

For deeper coverage, CLI commands, and exam-specific trivia, see the detailed reference:

- **SageMaker, Rekognition, Comprehend, and ML services**: [`AdvancedIdentity-DR-OtherServices.md`](../../detailed-reference/AdvancedIdentity-DR-OtherServices.md)

---

*Section 16 — Machine Learning | Last Validated: 2026-05-10*
