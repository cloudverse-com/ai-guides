# Advanced RAG with Azure AI Search
## Setup Guide for Multimodal Document Intelligence

> **Version 3 — March 2026**
> Changes from v2 are marked with `[v3]` inline.

---

## Overview

This guide covers deploying an advanced Retrieval-Augmented Generation (RAG) system using Azure AI Search with Document Intelligence, image verbalization, and vector embeddings — all powered through a single Microsoft Foundry project. No standalone Azure OpenAI resource is required.

**Key capabilities:**
- PDF document processing with layout understanding via Document Intelligence
- Image extraction and AI-powered verbalization using GPT-5 or GPT-5.4
- Vector embeddings using `text-embedding-3-large` deployed in Foundry
- Managed identity-based security (keyless authentication throughout)

---

## Architecture Overview

| Component | Azure Service | Purpose in RAG Pipeline |
|---|---|---|
| Azure AI Search | Search service | Hosts vector index, executes skillset, orchestrates enrichment pipeline |
| Azure AI Services (Multi-service) | AI Services account | Document Intelligence layout skill — extracts text, tables, layout, images from PDFs |
| Microsoft Foundry Project | Foundry project | Hosts `text-embedding-3-large` (vectorization) and GPT-5/5.4 (image verbalization) |
| Azure Blob Storage | Storage account | Source PDFs (input) and extracted images (output) |

> **Simplified Architecture**: Microsoft Foundry hosts both the embedding model and image verbalization model. A separate standalone Azure OpenAI resource is not required.

---

## Region Strategy — The Critical Constraint

### Rule 1: AI Search and AI Services MUST be in the same region

When using API key authentication (default), Azure AI Search and Azure AI Services must be in the **same Azure region**. Mismatched regions prevent the Document Intelligence skill from functioning in the indexer.

> **Exception**: Using Microsoft Entra ID (keyless) authentication (preview) relaxes this requirement. For wizard-based setup, same-region is required.

### Rule 2: AI Services must be in one of three wizard-supported regions

| Region | Doc Intelligence in Wizard | All AI Search Features | Notes |
|---|---|---|---|
| **East US** | Yes | Yes | US option |
| **West Europe** | Yes | Yes | EU option |
| **North Central US** | Yes | Yes | US alternative |
| East US 2 | No | Yes | Programmatic / Entra ID auth only |
| West US 2 | No | Yes | Programmatic / Entra ID auth only |
| All other regions | No | Varies | Check Azure region support table |

> **[v3]** The column previously labeled "Recommended" has been removed. Region selection depends on your capacity availability and requirements.

### Rule 3: Microsoft Foundry CAN be in a different region

> **[v3] Tested scenario (March 2026)**: AI Search and AI Services in **East US**, Foundry project in **East US 2** — fully operational with `text-embedding-3-large` and GPT-5.4 for image verbalization. Validated hands-on; other combinations may also work.

### Deployment Layout — Example Configuration

> **[v3]** The following is an **example** tested configuration as of March 2026, not a universal recommendation. Substitute regions based on your capacity availability, data residency, and compliance needs.

| Service | Example Region | Notes |
|---|---|---|
| Azure AI Search | East US (example) | Wizard-supported; must match AI Services region |
| Azure AI Services | East US (example) | Must match AI Search; wizard-supported for Doc Intelligence |
| Microsoft Foundry Project | East US 2 (example) | Full model availability: GPT-5, Claude, Mistral, `text-embedding-3-large` |
| Azure Blob Storage | Same as AI Search | Co-located with AI Search for indexer performance |

### Capacity Warning: East US and East US 2

Both East US and East US 2 frequently experience capacity constraints for Azure AI Search Basic and Standard tiers.

> **[v3] Important**: Your fallback region must still be one of the three wizard-supported regions (East US, West Europe, North Central US). Do NOT fall back to West US 2 or other non-wizard regions if using wizard-based Document Intelligence — those require the programmatic Entra ID approach instead.

**Options when your preferred region has no capacity:**
1. **Try North Central US** — wizard-supported; often has available capacity
2. **Try West Europe** — wizard-supported; good if data residency is flexible
3. **Use programmatic setup with Entra ID auth (preview)** — removes all region co-location requirements; AI Search can be in any region

---

## Azure AI Search Tier Requirements

| Tier | Doc Intelligence Skillset | Vector Search | Semantic Ranker | Multimodal Wizard | Monthly Cost |
|---|---|---|---|---|---|
| Free | Yes | Limited | Yes (1,000 req/month free) | No | $0 |
| Basic | Yes | Yes | Yes | Yes | ~$75 |
| Standard S1 | Yes | Yes | Yes | Yes | ~$250 |
| Standard S2 | Yes | Yes | Yes | Yes | ~$1,000 |

> **[v3] Semantic Ranker on Free tier**: Available with 1,000 requests/month at no charge — confirmed by real-world testing March 2026.

> **[v3] Free tier and Document Intelligence**: Free tier CAN create and run indexers with Document Intelligence skill — confirmed hands-on (March 2026). Limitations: 50 MB storage, 3 indexes, 3 indexers max.

The "Import and vectorize data" multimodal wizard requires **Basic tier or higher**. The standard "Import data" wizard works on Free tier and also exposes the Document Intelligence skill.

---

## Microsoft Foundry Model Deployments

> **[v3]** All references to "hub-based Foundry project" removed. Use the **new Microsoft Foundry project** (non-hub). Hub-based projects are deprecated — Microsoft is migrating customers to new Foundry projects. When the wizard shows "AI Foundry Hub catalog models" in a dropdown, this connects to a standard Foundry project via its endpoint, not a legacy hub.

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
| `gpt-5.4` | Yes | $$$$ | Set temperature=1.0 — see configuration below |
| `gpt-5` | Yes | $$$$ | Set temperature=1.0 — see configuration below |
| `gpt-5-mini` | Yes | $$ | Set temperature=1.0 — see configuration below |
| `gpt-4o` | Yes | $$$ | No special requirements |
| `gpt-4o-mini` | Yes | $ | No special requirements |
| `phi-4` | Yes | $ | Foundry project catalog path only |

### [v3] Setting temperature=1.0 for GPT-5 Models

Any GPT-5 series model (gpt-5, gpt-5.4, gpt-5-mini) used for image verbalization MUST have temperature set to 1.0. Any other value causes the skill to fail silently.

**Via wizard UI:**
1. In the Content Embedding step, after selecting your GPT-5 model
2. Find "Advanced settings" or "Skill parameters"
3. Set temperature to 1.0
4. If not exposed in wizard, use the programmatic approach below

**Via REST API or SDK — add modelParameters to your GenAI Prompt skill definition:**
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
```

Key field: `"modelParameters": { "temperature": 1.0 }` — required for all GPT-5 series models.

### Embedding Model Options

| Model | Dimensions | Best For | Notes |
|---|---|---|---|
| `text-embedding-3-large` | 3072 | English RAG (highest accuracy) | Recommended; deploy in Foundry project |
| `text-embedding-3-small` | 1536 | Cost-optimized English RAG | Lower accuracy |
| `Cohere-embed-v4` | 1024 | Multilingual RAG | Via Foundry project catalog vectorizer |
| `text-embedding-ada-002` | 1536 | Legacy only | Not recommended for new deployments |

---

## Managed Identity and RBAC Configuration

All connections use system-assigned managed identities. Never use connection strings or API keys in production.

### Step 1: Enable System-Assigned Managed Identity

Azure Portal → Resource → Identity → System assigned → On → Save

Enable on:
- Azure AI Search service
- Azure AI Services multi-service account (if using Entra ID cross-region auth)

### Step 2: Role Assignments

| Source (Needs Access) | Target Resource | Role to Assign | Scope |
|---|---|---|---|
| AI Search | Blob Storage | Storage Blob Data Reader | Source PDFs container |
| AI Search | Blob Storage | Storage Blob Data Contributor | Image output container |
| AI Search | Azure AI Services | Cognitive Services User | AI Services resource |
| AI Search | Foundry / Azure OpenAI | Cognitive Services OpenAI User | Foundry or OpenAI resource |
| Your app / user | AI Search | Search Index Data Reader | AI Search resource |
| Your app / user | AI Search | Search Index Data Contributor | AI Search resource |

### Step 3: How to Assign Roles in Azure Portal

1. Navigate to the target resource
2. Click Access Control (IAM)
3. Click + Add → Add role assignment
4. Select the role
5. Click Next → "Assign access to" → Managed identity
6. Click + Select members → select the AI Search managed identity
7. Click Review + assign
8. Allow 2–5 minutes for propagation before testing

### RBAC Roles Reference

| Role | What It Allows |
|---|---|
| Storage Blob Data Reader | Read blob content and metadata |
| Storage Blob Data Contributor | Read, write, delete blobs |
| Cognitive Services OpenAI User | Call Azure OpenAI/Foundry inference APIs |
| Cognitive Services User | Call Azure AI Services APIs (Document Intelligence) |
| Search Service Contributor | Create/manage indexes, indexers, skillsets |
| Search Index Data Contributor | Import and update documents in search indexes |
| Search Index Data Reader | Query search indexes from applications |

---

## Deployment Checklist

### Prerequisites

- [ ] Azure subscription with Owner or Contributor + User Access Administrator permissions
- [ ] Microsoft Foundry project created (new Foundry project — not legacy hub-based)
- [ ] Models deployed in Foundry: text-embedding-3-large + at least one vision model (gpt-5.4 or gpt-4o)
- [ ] Target region decided for AI Search and AI Services (must match; must be East US, West Europe, or North Central US for wizard)

### Resource Provisioning Order

**1. Create Azure AI Services (Multi-service account)**
- Region: East US, West Europe, or North Central US
- Enable system-assigned managed identity

**2. Create Azure AI Search**
- Region: Same as AI Services
- Tier: Free (dev/test) or Basic/Standard S1 (production)
- Enable system-assigned managed identity
- Enable RBAC: Settings → Keys → "Role-based access control" or "Both"
- If capacity unavailable: only fall back to another wizard-supported region (East US, West Europe, North Central US). Do NOT use West US 2 or other non-wizard regions if using wizard for Doc Intelligence.

**3. Create Azure Blob Storage Account**
- Region: Same as AI Search
- Create container for source PDFs (e.g., documents)
- Create container for extracted images (e.g., images)

**4. Configure RBAC Roles**
- Assign all roles per the table above
- Wait 2–5 minutes for propagation

**5. Upload Test Documents**
- Upload 2–3 test PDFs to source container

**6. Create Multimodal RAG Index** (see Wizard Walkthrough below)

---

## Wizard Configuration Walkthrough

### Which Wizard to Use

| Wizard | Access Point | Doc Intelligence | Embedding | Image Verbalization |
|---|---|---|---|---|
| Import data (classic) | AI Search portal → Import data | Yes (Free tier+) | No | No |
| Import and vectorize data | AI Search portal → Import and vectorize data | Yes | Yes | Yes |
| Multimodal RAG wizard | Foundry project → Multimodal RAG | Yes | Yes | Yes |

### Step-by-Step: Import and Vectorize Data Wizard

**Step 1 — Connect to your data**
- Data Source: Azure Blob Storage
- Authentication: Use managed identity (not connection string)
- Container: Source PDFs container

**Step 2 — Content extraction**
- Enable AI enrichment: Yes
- AI Services resource: Select your account (East US / West Europe / North Central US)
- Enable skill: Document Layout (Document Intelligence)

**Step 3 — Content embedding (choose one path)**

Path A — Azure OpenAI:
- Kind: Azure OpenAI → select your Azure OpenAI resource
- Image verbalization: gpt-5.4 or gpt-4o; Text vectorization: text-embedding-3-large

Path B — Foundry Project (recommended if using Foundry):
- Kind: "AI Foundry Hub catalog models" (Note: despite the label, this connects to your standard Foundry project — not a legacy hub)
- Select your Foundry project
- Image verbalization: gpt-5.4 or gpt-4o; Text vectorization: text-embedding-3-large
- If using GPT-5 model: set temperature=1.0 (see GPT-5 configuration section above)

**Step 4 — Image output**
- Enable image output: Yes
- Output container: Images Blob container
- Authentication: Managed identity

**Step 5 — Advanced settings**
- Chunk size: 2048 tokens | Overlap: 512 tokens
- Embedding dimensions: 3072
- Semantic ranker: Yes (available on all tiers including Free, 1,000 req/month free)

**Step 6 — Review and create**
- Set index name and indexer schedule, then click Create

---

## Troubleshooting Common Issues

| Issue | Likely Cause | Solution |
|---|---|---|
| Doc Intelligence not in wizard | AI Services not in East US / West Europe / North Central US | Redeploy AI Services to a wizard-supported region matching AI Search |
| AI Search + AI Services mismatch | Different regions with API key auth | Deploy both to same region, or use Entra ID auth (preview) |
| Basic tier unavailable — high demand | East US / East US 2 capacity | Try North Central US or West Europe (both wizard-supported). If neither available, use programmatic Entra ID approach |
| Indexer 403 on Blob Storage | Missing RBAC role | Assign Storage Blob Data Reader to AI Search managed identity on container |
| Embedding auth error | Missing RBAC role | Assign Cognitive Services OpenAI User to AI Search managed identity on Foundry resource |
| Image verbalization empty | GPT-5 temperature wrong | Set temperature=1.0 in GenAI Prompt skill modelParameters |
| Import and vectorize button missing | Free tier | Upgrade to Basic; Free only shows classic Import data wizard |
| Foundry models not in dropdown | Wrong project type | Use new Foundry project (not legacy hub); ensure project endpoint is linked |
| Cross-region latency on indexing | Foundry in different region | Expected and acceptable; indexing is scheduled, not real-time |

---

## Production Best Practices

### Security
- Use managed identities exclusively — never connection strings or API keys
- Enable Azure Private Link for sensitive workloads
- Enable diagnostic logging to Azure Monitor

### Performance
- Co-locate AI Search, AI Services, and Blob Storage in the same region
- Use Standard S1 or higher for production (Basic has 15 GB storage limit)
- Schedule indexers during off-peak hours for large document sets

### Cost Management
- Document Intelligence charges ~$0.01/page for layout model
- Use gpt-4o-mini for simple diagram verbalization instead of gpt-5.4
- Monitor Foundry token usage: Foundry portal → Operate → Cost analysis

### Monitoring
- Enable Application Insights on AI Search
- Review indexer execution history: AI Search → Indexers → [name] → Execution history
- Set alerts for indexer failures and quota limits

---

## References

1. Microsoft Learn. Azure AI Search — Supported Regions. https://learn.microsoft.com/en-us/azure/search/search-region-support
2. Microsoft Learn. Document Layout Skill. https://learn.microsoft.com/en-us/azure/search/cognitive-search-skill-document-intelligence-layout
3. Microsoft Learn. GenAI Prompt Skill. https://learn.microsoft.com/en-us/azure/search/cognitive-search-skill-genai-prompt
4. Microsoft Learn. Configure a Managed Identity — Azure AI Search. https://learn.microsoft.com/en-us/azure/search/search-how-to-managed-identities
5. Microsoft Learn. Connect to Azure Storage using a managed identity. https://learn.microsoft.com/en-us/azure/search/search-howto-managed-identities-storage
6. Microsoft Learn. Service Limits for Tiers and SKUs. https://learn.microsoft.com/en-us/azure/search/search-limits-quotas-capacity
7. Microsoft Learn. Integrated Vectorization with Models from Microsoft Foundry. https://learn.microsoft.com/en-us/azure/search/vector-search-integrated-vectorization-ai-studio
8. Microsoft Learn. Microsoft Foundry Model Catalog Vectorizer. https://learn.microsoft.com/en-us/azure/search/vector-search-vectorizer-azure-machine-learning-ai-studio-catalog
9. Microsoft Learn. Multimodal Search Concepts. https://learn.microsoft.com/en-us/azure/search/multimodal-search-overview
10. Microsoft Learn. Azure built-in roles. https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles
11. Microsoft Learn. Semantic Ranking Overview. https://learn.microsoft.com/en-us/azure/search/semantic-search-overview
12. Microsoft Learn. Migrate from Hub-based to Foundry Projects. https://learn.microsoft.com/en-us/azure/foundry-classic/how-to/migrate-project


