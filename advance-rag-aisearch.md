# Advanced RAG with Azure AI Search
## Setup Guide for Multimodal Document Intelligence

> **Version 3 — March 2026**
> Changes from v2 are marked with `[v3]` inline.

**Complete deployment guide for production-ready multimodal RAG architecture using Microsoft Foundry**

---

## Overview

This guide covers deploying an advanced Retrieval-Augmented Generation (RAG) system using Azure AI Search with Document Intelligence, image verbalization, and vector embeddings — all powered through a single Microsoft Foundry project. No standalone Azure OpenAI resource is required.

**Key capabilities enabled by this architecture:**
- PDF document processing with layout understanding via Document Intelligence
- Image extraction and AI-powered verbalization using GPT-5 or GPT-5.4
- Vector embeddings using `text-embedding-3-large` deployed in Foundry
- Managed identity-based security (keyless authentication throughout)
- Integration with Microsoft Foundry IQ for unified knowledge retrieval

---

## Architecture Overview

| Component | Azure Service | Purpose in RAG Pipeline |
|---|---|---|
| Azure AI Search | Search service | Hosts vector index, executes skillset, orchestrates enrichment pipeline |
| Azure AI Services (Multi-service) | AI Services account | Provides Document Intelligence layout skill for PDF extraction |
| Microsoft Foundry Project | Foundry project | Hosts `text-embedding-3-large` (vectorization) and GPT-5/5.4 (image verbalization) |
| Azure Blob Storage | Storage account | Stores source PDFs (input) and extracted images (output) |

> **Simplified Architecture**: Microsoft Foundry hosts both the embedding model and the image verbalization model. A separate standalone Azure OpenAI resource is not required.

---

## Region Strategy — The Critical Constraint

### Rule 1: AI Search and AI Services MUST be in the same region

When using API key authentication (default), the Azure AI Search service and the Azure AI Services multi-service account must be deployed in the **same Azure region**. Mismatched regions will prevent the Document Intelligence skill from functioning in the indexer.

> **Exception**: If you configure the connection using **Microsoft Entra ID (keyless) authentication** (currently in preview), the same-region requirement is relaxed. However, for wizard-based setup, same-region is required.

### Rule 2: AI Services must be in one of three wizard-supported regions

| Region | Doc Intelligence in Wizard | All AI Search Features | Notes |
|---|---|---|---|
| **East US** | ✅ Yes | ✅ Yes | US option |
| **West Europe** | ✅ Yes | ✅ Yes | EU option |
| **North Central US** | ✅ Yes | ✅ Yes | US alternative |
| East US 2 | ❌ No | ✅ Yes | Programmatic / Entra ID auth only |
| West US 2 | ❌ No | ✅ Yes | Programmatic / Entra ID auth only |
| All other regions | ❌ No | Varies | Check Azure region support table |

> **[v3]** The column previously labeled "Recommended" has been removed. Region selection depends on your capacity availability and requirements.

### Rule 3: Microsoft Foundry CAN be in a different region

> **[v3] Tested scenario (March 2026)**: AI Search and AI Services in **East US**, Foundry project in **East US 2** — fully operational with `text-embedding-3-large` and GPT-5.4 for image verbalization. This was validated hands-on; other region combinations may also work.

### Deployment Layout — Example Configuration

> **[v3]** The following is an **example** tested configuration as of March 2026, not a universal recommendation. Substitute regions based on your capacity availability, data residency requirements, and compliance constraints.

| Service | Example Region | Notes |
|---|---|---|
| Azure AI Search | East US *(example)* | Wizard-supported; must match AI Services region |
| Azure AI Services | East US *(example)* | Must match AI Search; wizard-supported for Doc Intelligence |
| Microsoft Foundry Project | East US 2 *(example)* | Full model availability: GPT-5, Claude, Mistral, `text-embedding-3-large` |
| Azure Blob Storage | Same as AI Search | Co-located with AI Search for indexer performance |

### ⚠️ Capacity Warning: East US and East US 2

Both East US and East US 2 frequently experience **capacity constraints** for Azure AI Search Basic and Standard tiers.

> **[v3] Important**: Your fallback region must still be one of the three wizard-supported regions **(East US, West Europe, North Central US)**. Do **not** fall back to West US 2 or other non-wizard regions if you need wizard-based Document Intelligence — those require the programmatic Entra ID approach instead.

**Options when your preferred region has no capacity:**
1. **Try North Central US** — wizard-supported; often has available capacity
2. **Try West Europe** — wizard-supported; good if data residency is flexible
3. **Use programmatic setup with Entra ID auth (preview)** — removes all region co-location requirements; AI Search can be in any region

---

## Azure AI Search Tier Requirements

| Tier | Doc Intelligence Skillset | Vector Search | Semantic Ranker | Multimodal Wizard | ~Monthly Cost |
|---|---|---|---|---|---|
| **Free** | ✅ Yes | Limited | ✅ Yes (1,000 req/month free) | ❌ No | $0 |
| **Basic** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ~$75 |
| **Standard S1** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ~$250 |
| **Standard S2** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ~$1,000 |

> **[v3] Semantic Ranker on Free tier**: Available with **1,000 requests/month** at no charge — confirmed by real-world testing (March 2026).

> **[v3] Free tier and Document Intelligence**: Free tier **can** create and run indexers with the Document Intelligence skill — confirmed by hands-on testing (March 2026). Limitations: 50 MB storage, 3 indexes, 3 indexers max.

**The "Import and vectorize data" (multimodal) wizard requires Basic tier or higher.** The standard "Import data" wizard works on Free tier and also exposes the Document Intelligence skill.

---

## Microsoft Foundry Model Deployments

> **[v3]** References to "hub-based Foundry project" have been removed throughout. Use the **new Microsoft Foundry project** (non-hub). Hub-based projects are being deprecated — Microsoft is actively migrating customers to the new Foundry project model. When the wizard shows **"AI Foundry Hub catalog models"** in a dropdown, this refers to connecting to a standard Foundry project via its endpoint, not a legacy hub.

### Required Model Deployments

| Model | Type | Deployment Type | Used For |
|---|---|---|---|
| `text-embedding-3-large` | Embedding | Global Standard | Text vectorization of document chunks |
| `gpt-5` or `gpt-5.4` | Chat completion (vision) | Global Standard | Image verbalization |

### Optional Additional Models

| Model | Used For |
|---|---|
| `gpt-4o` or `gpt-4o-mini` | Alternative image verbalization (lower cost) |
| `claude-sonnet-4-6` | Agent reasoning in Foundry IQ |
| `mistral-document-ai-2505` | Alternative document extraction |

### Image Verbalization Model Options

| Model | Vision | Cost Profile | Special Requirement |
|---|---|---|---|
| `gpt-5.4` | ✅ | $$$$ | **Set `temperature=1.0`** — see configuration below |
| `gpt-5` | ✅ | $$$$ | **Set `temperature=1.0`** — see configuration below |
| `gpt-5-mini` | ✅ | $$ | **Set `temperature=1.0`** — see configuration below |
| `gpt-4o` | ✅ | $$$ | No special requirements |
| `gpt-4o-mini` | ✅ | $ | No special requirements |
| `phi-4` | ✅ | $ | Foundry project catalog path only |

### [v3] Setting temperature=1.0 for GPT-5 Models

Any GPT-5 series model (`gpt-5`, `gpt-5.4`, `gpt-5-mini`) used for image verbalization **must** have `temperature` set to `1.0`. Any other value causes the skill to fail silently.

**Via wizard UI:**
1. In the Content Embedding step, after selecting your GPT-5 model
2. Find **"Advanced settings"** or **"Skill parameters"**
3. Set `temperature` to `1.0`
4. If not exposed in wizard, use the programmatic approach below

**Via REST API / SDK — add `modelParameters` to your GenAI Prompt skill:**

```json
{
  "@odata.type": "#Microsoft.Skills.Text.GenAIPromptSkill",
  "name": "image_verbalization",
  "context": "/document/normalized_images/*",
  "inputs": [
    { "name": "image", "source": "/document/normalized_images/*/data" }
  ],
  "outputs": [
    { "name": "text", "targetName": "imageDescription" }
  ],
  "modelParameters": {
    "temperature": 1.0,
    "max_tokens": 800
  }
}
