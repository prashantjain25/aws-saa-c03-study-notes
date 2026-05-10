# Section 17 — Other Services

> **Purpose**: AWS offers 200+ services. While a solutions architect cannot be an expert in all of them, they must recognize when a specialized service is the right tool for a specific problem. This section covers services that do not fit neatly into the primary categories but frequently appear in architecture decisions and certification exams.
>
> **Official Documentation**: [WorkSpaces](https://docs.aws.amazon.com/workspaces/) | [AppStream](https://docs.aws.amazon.com/appstream2/) | [IoT Core](https://docs.aws.amazon.com/iot/) | [Amplify](https://docs.aws.amazon.com/amplify/)

---

## 1. End-User Computing

### 1.1 Amazon WorkSpaces

Managed **Desktop-as-a-Service (DaaS)**:
- Persistent virtual desktops (Windows or Linux)
- Each user gets a dedicated volume (data persists between sessions)
- Integrates with Active Directory (AWS Managed Microsoft AD or on-prem AD)
- **Use case**: Remote workers, contractors, secure access to sensitive applications

### 1.2 Amazon AppStream 2.0

Managed **application streaming** (non-persistent):
- Applications stream to users; no persistent desktop
- User data can be saved to S3 or home folders
- **Use case**: Software trials, classroom labs, temporary access to high-end applications

> **WorkSpaces vs AppStream**: WorkSpaces = persistent desktop (like your laptop in the cloud). AppStream = streamed application (like Citrix). Use WorkSpaces for daily work; AppStream for occasional application access.

---

## 2. Internet of Things (IoT)

### 2.1 AWS IoT Core

Managed MQTT broker for IoT device communication:
- **Device Gateway**: MQTT, WebSocket, HTTP protocols
- **Device Shadows**: JSON documents representing device state (desired vs reported)
- **Rules Engine**: Route messages to S3, Lambda, Kinesis, SNS, etc.
- **Fleet Provisioning**: Automated device onboarding at scale
- **Device Defender**: ML-based anomaly detection for device behavior

**IoT architecture pattern**:
```
IoT Devices ──► IoT Core (MQTT) ──► Rules Engine ──► Kinesis ──► Analytics
                                    │
                                    └──► Lambda ──► DynamoDB (device state)
                                    └──► S3 (raw telemetry archival)
```

---

## 3. Developer Tools

### 3.1 AWS Amplify

Full-stack development platform for web and mobile:
- **Frontend libraries**: React, Vue, Angular, iOS, Android, Flutter
- **Backend**: Auto-provision auth (Cognito), API (AppSync/API Gateway), storage (S3), functions (Lambda)
- **Hosting**: CI/CD-connected hosting with CloudFront CDN
- **Use case**: Rapid prototyping, full-stack serverless applications

### 3.2 AWS AppSync

Managed GraphQL API service:
- **Real-time subscriptions**: WebSocket-based push updates
- **Offline support**: Datastore sync for mobile apps
- **Multiple data sources**: DynamoDB, Lambda, HTTP, RDS, OpenSearch
- **Use case**: Mobile backends, real-time dashboards, social apps

### 3.3 AWS Device Farm

Cloud-based mobile testing:
- Test on real devices (not emulators)
- Android, iOS, Fire OS
- Automated testing with Appium, Espresso, XCTest

---

## 4. Satellite and Edge

### 4.1 AWS Ground Station

Satellite communication as a service:
- Rent antenna access by the minute
- Download satellite data directly to S3/VPC
- **Use case**: Earth observation, weather monitoring, satellite imagery companies

### 4.2 AWS Wavelength and Local Zones

**Wavelength**: AWS infrastructure embedded in 5G telecom provider data centers. Single-digit millisecond latency to mobile devices.

**Local Zones**: AWS infrastructure in metropolitan areas (not full regions). Low latency for applications requiring proximity to end-users (gaming, media, AR/VR).

---

## 5. Contact Center

### 5.1 Amazon Connect

Cloud contact center:
- **Omnichannel**: Voice, chat, SMS, email
- **Flows**: Visual IVR designer
- **AI integration**: Transcribe (speech-to-text), Comprehend (sentiment), Lex (chatbots)
- **Pricing**: Pay per minute of usage. No long-term contracts.

---

## 6. Blockchain

### 6.1 Amazon Managed Blockchain

Managed Hyperledger Fabric or Ethereum networks:
- **Use case**: Supply chain tracking, cross-organizational transaction ledgers, consortia networks
- **Reality check**: Most "blockchain" use cases are better served by DynamoDB with cross-account access + CloudTrail for audit. Evaluate whether immutability and distributed consensus are actually required.

---

## 7. Quantum Computing

### 7.1 Amazon Braket

Managed quantum computing platform:
- Access to quantum hardware (D-Wave, IonQ, Rigetti) and simulators
- **Use case**: Research, experimentation, quantum algorithm development
- **Reality check**: Quantum computing is not yet practical for production business applications. Use for research and education only.

---

## 8. Points to Remember

- **WorkSpaces = persistent DaaS; AppStream = non-persistent app streaming**.
- **IoT Core provides MQTT broker, device shadows, and rules engine** — the backbone of IoT on AWS.
- **Amplify accelerates full-stack serverless development** — not for large enterprise applications requiring custom infrastructure.
- **AppSync provides managed GraphQL with real-time subscriptions** — ideal for mobile and real-time applications.
- **Local Zones and Wavelength provide low-latency edge compute** — for gaming, media, and 5G applications.
- **Amazon Connect is a pay-per-use contact center** — no infrastructure commitment.
- **Managed Blockchain serves specific consortia use cases** — most applications do not need blockchain.
- **Braket is for quantum research** — not production workloads.

---

*Section 17 — Other Services | Last Validated: 2026-05-10*
