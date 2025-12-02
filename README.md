# Legal Document Classification Project

We are struggling with high-variance and false-positive classifications when automatically identifying key legal documents (Summons and Complaint) within diverse, multi-page PDF packets from various counties. Our current purely semantic (Vector DB) approach fails to reliably distinguish between a core pleading (e.g., a Complaint) and a similar-looking administrative document (e.g., an Affidavit of Service, a Judgment Order) due to shared legal vocabulary. This leads to inefficient downstream processing and high human audit costs. We are solving this now to reduce the manual classification burden on our processing team and increase the velocity of case intake.

## Problem Statement
We are building a system to automatically classify legal documents, specifically **Summons** and **Complaints**. Currently, legal teams manually review PDFs to determine document type, which is time-consuming and error-prone. Automating this will improve processing speed, reduce errors, and enable downstream search and retrieval systems.

---

## Goals

### Software Goals

1.  **Develop a Hybrid Scoring Function:** Create a system, `classify_page_hybrid`, that combines semantic similarity (K-Nearest Neighbors/KNN) with weighted, external Rule-Based Biases (loaded from a dynamic JSON configuration).
2.  **Implement Sequential Memory:** Incorporate a memory state (`last_page_label`) into the processing pipeline to apply a strong **Sequence Bias** based on expected document flow (e.g., Summons $\rightarrow$ Complaint continuation).
3.  **Enable Dynamic Rule Management:** Decouple keyword/weight tuning from code deployment by loading all bias logic from an external JSON file (`bias_config.json`).
4.  **Create a Fine-Tuning Agent:** Utilize the existing LLM API (`gpt-4o-mini`) to generate a prioritized list of candidate rules and weights for rapid, data-driven updates to the JSON configuration for new county formats.

### Metric Goals

* **Improve F1-Score:** Increase the F1-Score for the target classes (**'summons'** and **'complaint'**) by **10%** over the Vector-only Baseline.
* **Improve Precision for Exclusion Classes:** Achieve **95% Precision** for the **'noise'** class (ensuring fewer supporting documents are misclassified as core pleadings).
* **Tradeoff:** Increase recall without decreasing precision by more than **5%**.
---

## Experiment Design

**Baseline:** Current production solution utilizing **LegalBERT + ChromaDB (Vector-only KNN retrieval)** with a simple threshold for classification.

### Data Motivation

The high false-positive rate for core pleadings (e.g., classifying a **Judgment Order** as a **Complaint**, or an **Affidavit** as a **Summons**) proves that semantic similarity alone is insufficient for this domain. The project is validated by the need to enforce hard structural rules (like header exclusions and sequence) which the current system ignores.

### Hypothesis 1: Hybrid Fusion Will Outperform Vector-Only Baseline

* **Method:** Implement the initial version of `classify_page_hybrid` combining semantic scores and rule-based scores. Initially focus on implementing the **Kill Switches** (Negative Biases) to enforce precision.
* **Metric:** Precision and F1-Score for the **'noise'** class.
* **Success Criteria:** **Precision for 'noise' > 90%**. This proves the exclusion rules work.
* **Timebox:** 5 days (Includes JSON creation and initial Python implementation).
* **Failure Next Steps:** Re-evaluate the negative weights (reduce/increase penalties) or expand the list of Noise keywords in the JSON.
* **Success Next Steps:** Proceed to Hypothesis 2 to introduce sequential memory.

### Hypothesis 2: Sequential Memory Will Improve Section Boundary Detection

* **Method:** Integrate the `last_page_label` logic into the processing loop and apply a **Sequence Bias Score** (e.g., +0.6) for expected document continuations (`Summons` $\rightarrow$ `Summons`, `Complaint` $\rightarrow$ `Complaint`).
* **Metric:** F1-Score for **'summons'** and **'complaint'**.
* **Success Criteria:** F1-Score for both classes improves by **> 5%** compared to Hypothesis 1 results.
* **Timebox:** 5 days.
* **Failure Next Steps:** Adjust the `SEQUENCE_BONUS` weight or refine the logic for boundary termination (i.e., when a document *must* stop).
* **Success Next Steps:** Proceed to the Walk stage.

---

## Software Design

### System Components and Data Flow

| Component | Role in Hybrid System |
| :--- | :--- |
| **ChromaDB** | Stores all historical page embeddings and labels (Semantic Ground Truth). |
| **`bias_config.json`** | Defines all structural rules, keyword weights, and **Dynamic Header Zones**. |
| **`process_documents`** | Orchestrates the flow, maintains **Sequence Memory** (`last_page_label`), and writes audit logs. |
| **`classify_page_hybrid`** | The core decision function. Fuses $\text{Vector Score}$, $\text{Keyword Bias}$, and $\text{Sequence Bias}$ to produce the final classification. |

### $\text{classify-page-hybrid}$ Architecture

The final decision is a weighted fusion, prioritizing the KNN semantic context:

$$\text{Final Score} = (\text{Vector Score} \times W_V) + (\text{Bias Score} \times W_B)$$

Where:
* cosine score is the weight for **KNN Score (Major Contributor)** (e.g., 0.8 out of 1).
* bias weight modified by llm is the weight for **Rule/Sequence Bias (Tiebreaker)** (e.g., 0.2).
* total score is the sum of both scores. 

---

## Considerations

| Area | Description |
| :--- | :--- |
| **Success Criteria to Launch** | accuracy for Summons and Complaint must be $\mathbf{\geq 0.90}$ on the Holdout Validation set, and Noise Precision must be $\mathbf{\geq 0.95}$. |
| **What Could Go Wrong?** | **Vector Drift:** Degradation of KNN matches due to new document styles. **Mitigation:** avoid adding unchecked embeddings, only add llm checked embeddings |
| **Monitoring** | Monitor confidence scores per document. If average confidence for a class drops below a defined threshold (e.g., 0.6), trigger an alert to review the county's JSON configuration. |
| **Roll Back** | Revert to the last stable **`bias_config.json`** and the Vector-only KNN baseline. |
| **Security/Privacy** | This project deals only with metadata (embeddings) and classification logic (keywords/weights). No PII is stored or processed by the model itself. |

---
