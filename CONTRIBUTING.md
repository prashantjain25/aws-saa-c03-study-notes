# Contributing to AWS Solutions Architect Notes

Thank you for contributing to the AWS Solutions Architect Notes repository. This project is a curated, high-quality, production-grade reference for senior cloud architects, principal engineers, and AWS interviewers. 

To maintain the quality, consistency, and professional depth of this repository, all contributions must adhere to the guidelines outlined below.

---

## 📂 Repository Structure

The repository is organized into three distinct layers:

1. **`handbook/` (The Solutions Architect Handbook)**: High-level architectural reference sections (e.g., identity, database, compute, networking). Each section focuses on "why" before "what," trade-off analysis, operational realities, and failure modes.
2. **`original-notes/` (Study Notes)**: Granular, service-specific files that contain deep-dive CLI commands, exam-specific trivia, and niche service descriptions (e.g., Snowball, App Runner) structured around a strict 4-layer template.
3. **`reference/` (Reference Materials)**: Quick-reference cards, service comparison cheat sheets, and common command patterns.

---

## 📐 Formatting Rules for `original-notes/`

Every file under `original-notes/` must be structured into logical topics, and each topic **must** contain exactly the following sections in this sequence:

### 1. `## Topic Name`
A clear, engineering-focused topic heading.

### 2. `### 📖 Technical Specifications & AWS Core Concepts`
Deep-dive technical primitives, rules, constraints, limits, and service specifications. Skip textbook definitions; prioritize concrete facts (e.g., "S3 object size limit is 5 TB").

### 3. `### 🗺️ Visual Architecture: Title`
An inline **Mermaid** diagram visualizing the process flow, routing logic, or high-availability architecture of the topic.
- Use flowcharts (`graph TD` or `graph LR`) or sequence diagrams.
- Quote node labels containing special characters: `id["Label (With Parens)"]`.
- Do not use raw HTML tags inside node labels.
- Keep diagrams simple, clean, and directly aligned with the technical text.

### 4. `### 🧠 Architectural Probing & Decision Scenarios`
Practical, interview-level architectural challenges written in the **Ultra-Concise Scenario/Design** format (see details below).

### 5. `### 📐 Application Design Patterns & Trade-offs`
A critical analysis of different implementation patterns (e.g., Multi-Region Active-Active vs. Active-Passive) highlighting operational complexity, sync lag, cost, and latency trade-offs.

### 6. `### 🚀 Real-World Production Insights`
Real-world operational realities or "Battle Scares"—including common failure modes, API throttling issues, quota limits, partition limits, and outages encountered in real production environments.

### 7. `### 💻 Hands-on CLI Commands`
Concrete AWS CLI examples showing how to perform tasks related to the topic. All commands must include realistic parameters, query filters, and clean formatting.

---

## 🧠 The "Ultra-Concise" Scenario/Design Format

The **Architectural Probing & Decision Scenarios** section of note files and the **Architect Interview Challenges** section of handbook files must strictly follow this syntax:

```markdown
* **Scenario:** [Engineering constraint or problem statement].
  * **Design:** [Recommended architecture/solution]. Because [Direct technical rationale citing performance, cost, limit, or operational benefit].
```

### Writing Philosophy & Tone Guidelines

1. **Fluff-Free & Engineering-First**: Do not use corporate roleplay or narrative setups. Do not write "Your CISO mandates...", "Company X needs...", or "As a Solutions Architect, you must...". State the engineering requirement directly.
2. **No Generic Textbook Definitions**: Do not define what a service is. Focus on the *architectural decision* and the *technical trade-off*.
3. **Justification is Key**: The `Because` clause must justify the design using hard facts, such as AWS service limits, routing behaviors, cost differences, or performance metrics.

#### ❌ Incorrect (Too lengthy, roleplaying, generic rationale):
> * **Scenario:** Your Chief Information Security Officer (CISO) mandates that all incoming internet traffic must be deeply inspected by a fleet of third-party Palo Alto virtual firewalls before it is allowed to reach your application subnets.
>   * **Design:** We will use a Gateway Load Balancer to route traffic to the firewalls. Because this satisfies the security requirements of the CISO and ensures our applications are safe.

####  Correct (Direct, concise, technically rigorous):
> * **Scenario:** All incoming traffic must be deeply inspected by third-party virtual firewalls before reaching application subnets.
>   * **Design:** Route traffic from an internet-facing ALB through Gateway Load Balancer (GWLB) to a fleet of Palo Alto virtual firewalls before routing to the application subnets. Because GWLB abstracts fleet scaling, health checking, and transparently forwards traffic without complex NAT/proxy scripting.

---

## 🎨 Diagram & Visual Guidelines

Diagrams should be used **judiciously**. Only use a diagram if a process flow, routing path, or multi-account structure is complex enough to benefit from visual aid. 

- Keep the diagrams clean and avoid clutter.
- Focus on the decision points and actual architecture components.
- Always verify your Mermaid syntax locally before submitting a Pull Request.

---

## 🤝 Contribution Workflow

1. **Fork & Branch**: Create a feature branch off of `main` (e.g., `feature/cognito-security-notes`).
2. **Implement & Refactor**: Write your markdown following the guidelines above. If you modify an existing file, ensure you do not break the template structure.
3. **Verify Mermaid**: Render your Mermaid diagrams to ensure there are no syntax errors.
4. **Commit**: Keep commit messages clear and descriptive (e.g., `Add S3 lifecycle transition scenarios to Storage notes`).
5. **Submit a Pull Request**: Provide a short description of the changes you have introduced.
