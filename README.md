# dino.finder
A RAG based chat model hosted in AWS that helps answer dinosaur questions
# 🦕 Dino Finder - RAG Architecture with Pinecone & Amazon Bedrock

## Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          USER INTERACTION LAYER                              │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────┐
    │      Web Browser             │
    │  ┌────────────────────────┐  │
    │  │   React Frontend       │  │
    │  │   - TypeScript         │  │
    │  │   - WebSocket Client   │  │
    │  │   - Chat UI            │  │
    │  └────────────────────────┘  │
    └───────────┬──────────────────┘
                │
                │ WSS:// (WebSocket Secure)
                │ Question: "How tall was T-Rex?"
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          AWS CLOUD INFRASTRUCTURE                            │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌────────────────────────────────────────────┐
    │   API Gateway (WebSocket API)              │
    │   ┌──────────────────────────────────┐     │
    │   │  Routes:                         │     │
    │   │  • $connect                      │     │
    │   │  • $disconnect                   │     │
    │   │  • sendMessage ◄─────────────────┼─────┼── User Message Entry Point
    │   └──────────────────────────────────┘     │
    └───────────┬────────────────────────────────┘
                │
                │ Lambda Proxy Integration
                │
                ▼
    ┌────────────────────────────────────────────────────────┐
    │             AWS Lambda Function                         │
    │         (dino-finder-websocket-handler)                 │
    │  ┌──────────────────────────────────────────────────┐  │
    │  │  1. Validate API Key                             │  │
    │  │  2. Check Rate Limit                             │  │
    │  │  3. Extract User Question                        │  │
    │  └──────────────────────────────────────────────────┘  │
    └───────────┬────────────────────────────────────────────┘
                │
                │ User Question
                ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                    RAG RETRIEVAL PROCESS                         │
    └─────────────────────────────────────────────────────────────────┘
                │
                │ Step 1: Convert Question to Vector
                ▼
    ┌────────────────────────────────────────────┐
    │        Amazon Bedrock                      │
    │   ┌────────────────────────────────┐       │
    │   │  Titan Embeddings Model        │       │
    │   │  amazon.titan-embed-text-v1    │       │
    │   │                                │       │
    │   │  Input: "How tall was T-Rex?"  │       │
    │   │  Output: [0.234, -0.123, ...]  │       │ ◄── 1536-dimension vector
    │   │         (1536 dimensions)       │       │
    │   └────────────────────────────────┘       │
    └────────────────────────────────────────────┘
                │
                │ Question Embedding Vector
                │
                ▼
    ┌─────────────────────────────────────────────────────────┐
    │                   Pinecone Vector Database               │
    │                  (External SaaS Service)                 │
    │  ┌───────────────────────────────────────────────────┐  │
    │  │  Index: "dino-finder"                             │  │
    │  │  Dimension: 1536                                  │  │
    │  │  Metric: Cosine Similarity                        │  │
    │  │                                                    │  │
    │  │  ┌─────────────────────────────────────────────┐  │  │
    │  │  │  Vector Search Query                        │  │  │
    │  │  │  1. Compare question vector with all stored │  │  │
    │  │  │  2. Find top 3 most similar chunks          │  │  │
    │  │  │  3. Return matching text + metadata         │  │  │
    │  │  └─────────────────────────────────────────────┘  │  │
    │  │                                                    │  │
    │  │  Stored Vectors (from Kaggle data):               │  │
    │  │  ┌──────────────────────────────────────┐         │  │
    │  │  │ ID: "t-rex"                          │         │  │
    │  │  │ Vector: [0.245, -0.131, ...]         │         │  │
    │  │  │ Text: "T-Rex stood 12-20 feet tall..." │       │  │
    │  │  │ Metadata: {period: "Cretaceous"}     │         │  │
    │  │  └──────────────────────────────────────┘         │  │
    │  │  ┌──────────────────────────────────────┐         │  │
    │  │  │ ID: "triceratops"                    │         │  │
    │  │  │ Vector: [0.156, 0.234, ...]          │         │  │
    │  │  │ Text: "Triceratops had 800 teeth..." │         │  │
    │  │  └──────────────────────────────────────┘         │  │
    │  │  ... (1000s more dinosaur records)                │  │
    │  └───────────────────────────────────────────────────┘  │
    └─────────────────────────────────────────────────────────┘
                │
                │ Top 3 Relevant Context Chunks
                │ ┌────────────────────────────────────────┐
                │ │ 1. "T-Rex stood 12-20 feet tall..."    │
                │ │ 2. "T-Rex weighed 8-14 tons..."        │
                │ │ 3. "T-Rex lived in Cretaceous..."      │
                │ └────────────────────────────────────────┘
                ▼
    ┌─────────────────────────────────────────────┐
    │         AWS Lambda (Continued)               │
    │  ┌───────────────────────────────────────┐  │
    │  │  Build Augmented Prompt:              │  │
    │  │                                       │  │
    │  │  "Use this verified information:      │  │
    │  │   - T-Rex stood 12-20 feet tall...    │  │
    │  │   - T-Rex weighed 8-14 tons...        │  │
    │  │   - T-Rex lived in Cretaceous...      │  │
    │  │                                       │  │
    │  │  Question: How tall was T-Rex?"       │  │
    │  └───────────────────────────────────────┘  │
    └─────────────────────────────────────────────┘
                │
                │ Augmented Prompt with Context
                ▼
    ┌─────────────────────────────────────────────────────────┐
    │              Amazon Bedrock                              │
    │   ┌──────────────────────────────────────────────────┐  │
    │   │  Claude 3 Haiku (LLM)                            │  │
    │   │  anthropic.claude-3-haiku-20240307-v1:0          │  │
    │   │                                                   │  │
    │   │  Input: Augmented prompt with context            │  │
    │   │  Processing:                                      │  │
    │   │  1. Understand question                          │  │
    │   │  2. Analyze provided context                     │  │
    │   │  3. Generate accurate answer                     │  │
    │   │                                                   │  │
    │   │  Output: "Based on fossil evidence, T-Rex        │  │
    │   │  typically stood between 12-20 feet tall at      │  │
    │   │  the hip and weighed around 8-14 tons..."        │  │
    │   └──────────────────────────────────────────────────┘  │
    └─────────────────────────────────────────────────────────┘
                │
                │ Generated Answer
                ▼
    ┌────────────────────────────────────────────┐
    │         AWS Lambda (Final Step)             │
    │  ┌──────────────────────────────────────┐  │
    │  │  Send Response via WebSocket         │  │
    │  │  to API Gateway Management API       │  │
    │  └──────────────────────────────────────┘  │
    └───────────┬────────────────────────────────┘
                │
                │ Response Message
                ▼
    ┌────────────────────────────────────────────┐
    │   API Gateway (WebSocket API)              │
    │   - Routes response to connected client    │
    └───────────┬────────────────────────────────┘
                │
                │ WebSocket Message
                ▼
    ┌──────────────────────────────┐
    │      Web Browser             │
    │  ┌────────────────────────┐  │
    │  │   Display Answer:      │  │
    │  │   "T-Rex stood 12-20   │  │
    │  │   feet tall..."        │  │
    │  └────────────────────────┘  │
    └──────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                    SUPPORTING INFRASTRUCTURE                                 │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
    │   IAM Roles          │    │   CloudWatch Logs    │    │  Cost Explorer  │
    │                      │    │                      │    │                 │
    │  • Lambda Execution  │    │  • Lambda logs       │    │  • Bedrock cost │
    │  • Bedrock Access    │    │  • API Gateway logs  │    │  • Lambda cost  │
    │  • API Gateway Mgmt  │    │  • Error tracking    │    │  • Pinecone cost│
    └──────────────────────┘    └──────────────────────┘    └─────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATA PREPARATION PIPELINE (One-time)                      │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌───────────────┐
    │    Kaggle     │
    │  Dinosaur     │
    │  Dataset      │
    │  (CSV/JSON)   │
    └───────┬───────┘
            │
            │ Download
            ▼
    ┌───────────────────────┐
    │  Local Processing     │
    │  (Python Script)      │
    │  • Parse CSV          │
    │  • Create text chunks │
    │  • Generate metadata  │
    └───────┬───────────────┘
            │
            │ Processed chunks
            ▼
    ┌──────────────────────────┐
    │  Amazon Bedrock          │
    │  Titan Embeddings        │
    │  • Convert each chunk    │
    │    to 1536-dim vector    │
    └───────┬──────────────────┘
            │
            │ Embeddings + Text
            ▼
    ┌──────────────────────────┐
    │  Upload to Pinecone      │
    │  • Store vectors         │
    │  • Store text metadata   │
    │  • Enable fast search    │
    └──────────────────────────┘
```

---

## Data Flow Sequence Diagram

```
User          Browser       API Gateway    Lambda         Bedrock(Titan)   Pinecone    Bedrock(Claude)
 │               │               │            │                  │             │              │
 │  Type Q       │               │            │                  │             │              │
 │──────────────>│               │            │                  │             │              │
 │               │  WSS Connect  │            │                  │             │              │
 │               │──────────────>│            │                  │             │              │
 │               │               │  Invoke    │                  │             │              │
 │               │               │───────────>│                  │             │              │
 │               │               │            │  Create Embedding│             │              │
 │               │               │            │─────────────────>│             │              │
 │               │               │            │  [0.234, -0.12..]│             │              │
 │               │               │            │<─────────────────│             │              │
 │               │               │            │                  │             │              │
 │               │               │            │  Query(vector, top_k=3)        │              │
 │               │               │            │────────────────────────────────>│              │
 │               │               │            │  [Match1, Match2, Match3]      │              │
 │               │               │            │<────────────────────────────────│              │
 │               │               │            │                  │             │              │
 │               │               │            │  Generate(prompt + context)    │              │
 │               │               │            │────────────────────────────────────────────────>│
 │               │               │            │              Answer             │              │
 │               │               │            │<────────────────────────────────────────────────│
 │               │               │  Response  │                  │             │              │
 │               │               │<───────────│                  │             │              │
 │               │  WSS Message  │            │                  │             │              │
 │               │<──────────────│            │                  │             │              │
 │  Display      │               │            │                  │             │              │
 │<──────────────│               │            │                  │             │              │
 │               │               │            │                  │             │              │
```

---

## Component Details

### 1. **Frontend (Browser)**
- **Technology**: React + TypeScript + Vite
- **Responsibility**: 
  - User interface for chat
  - WebSocket connection management
  - Message display and interaction
- **Files**: `src/App.tsx`, `src/main.tsx`

### 2. **API Gateway WebSocket API**
- **AWS Service**: Amazon API Gateway V2
- **Responsibility**:
  - Handle WebSocket connections
  - Route messages to Lambda
  - Manage connection state
- **Routes**: `$connect`, `$disconnect`, `sendMessage`
- **File**: `aws-backend/cloudformation-template.yaml`

### 3. **AWS Lambda Function**
- **Runtime**: Python 3.9
- **Responsibility**:
  - Request validation (API key, rate limiting)
  - Orchestrate RAG pipeline
  - Call Bedrock for embeddings and generation
  - Query Pinecone for relevant context
  - Send responses back via WebSocket
- **File**: `aws-backend/lambda_handler.py`
- **Key Functions**:
  - `lambda_handler()` - Entry point
  - `search_knowledge_base()` - RAG retrieval
  - `generate_ai_response()` - LLM generation

### 4. **Amazon Bedrock - Titan Embeddings**
- **Model**: `amazon.titan-embed-text-v1`
- **Responsibility**:
  - Convert text to 1536-dimensional vectors
  - Used for both:
    - Initial data preparation (Kaggle → Pinecone)
    - Runtime query embedding
- **Input**: Text string (up to 8K tokens)
- **Output**: Float array [1536 dimensions]
- **Cost**: $0.0001 per 1K tokens (~$0.10 per 1M tokens)

### 5. **Pinecone Vector Database**
- **Type**: External SaaS (Managed Vector DB)
- **Responsibility**:
  - Store dinosaur data embeddings
  - Perform fast similarity search
  - Return most relevant context chunks
- **Configuration**:
  - Index: `dino-finder`
  - Dimension: 1536
  - Metric: Cosine similarity
  - Namespace: Optional for multi-tenancy
- **Free Tier**: 1 index, 100K vectors, 100K operations/month

### 6. **Amazon Bedrock - Claude 3 Haiku**
- **Model**: `anthropic.claude-3-haiku-20240307-v1:0`
- **Responsibility**:
  - Generate natural language answers
  - Use retrieved context from Pinecone
  - Provide accurate, contextual responses
- **Input**: Augmented prompt (question + context)
- **Output**: Conversational answer
- **Cost**: 
  - Input: $0.25 per 1M tokens
  - Output: $1.25 per 1M tokens

### 7. **IAM Roles & Permissions**
```yaml
LambdaExecutionRole:
  Permissions:
    - CloudWatch Logs (write)
    - Bedrock InvokeModel (both Titan & Claude)
    - API Gateway ManageConnections
    - (No Pinecone - uses API key)
```

---

## RAG Process Breakdown

### **Step 1: User Asks Question**
```
User Input: "How tall was T-Rex?"
```

### **Step 2: Create Query Embedding**
```python
# Lambda calls Bedrock Titan
response = bedrock.invoke_model(
    modelId='amazon.titan-embed-text-v1',
    body=json.dumps({"inputText": "How tall was T-Rex?"})
)
query_vector = [0.234, -0.123, 0.456, ...]  # 1536 numbers
```

### **Step 3: Search Pinecone**
```python
# Lambda queries Pinecone
results = pinecone_index.query(
    vector=query_vector,
    top_k=3,
    include_metadata=True
)

# Returns top 3 matches:
matches = [
    {
        "score": 0.92,
        "text": "T-Rex stood 12-20 feet tall at the hip and weighed 8-14 tons.",
        "metadata": {"period": "Cretaceous", "diet": "Carnivore"}
    },
    {
        "score": 0.87,
        "text": "T-Rex had powerful hind legs and could run up to 25 mph.",
        "metadata": {"period": "Cretaceous", "diet": "Carnivore"}
    },
    {
        "score": 0.81,
        "text": "T-Rex lived 68-66 million years ago in North America.",
        "metadata": {"period": "Cretaceous", "diet": "Carnivore"}
    }
]
```

### **Step 4: Build Augmented Prompt**
```python
context = "\n".join([match['text'] for match in matches])

prompt = f"""You are Dino Finder, an expert on dinosaurs.

Use ONLY the following verified information to answer:

{context}

Question: How tall was T-Rex?

Answer based only on the information above."""
```

### **Step 5: Generate Answer with Claude**
```python
response = bedrock.invoke_model(
    modelId='anthropic.claude-3-haiku-20240307-v1:0',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1000,
        "messages": [{"role": "user", "content": prompt}]
    })
)

answer = "Based on fossil evidence, Tyrannosaurus Rex stood approximately 
12-20 feet tall at the hip and weighed between 8-14 tons. This massive 
predator lived during the Late Cretaceous period, 68-66 million years ago."
```

### **Step 6: Return to User**
```python
# Lambda sends via WebSocket
apigateway_management.post_to_connection(
    ConnectionId=connection_id,
    Data=json.dumps({"message": answer})
)
```

---

## Cost Analysis (RAG-Enabled)

### Per Query Costs:
| Service | Operation | Cost per 1000 Queries |
|---------|-----------|----------------------|
| API Gateway | WebSocket messages | $0.001 |
| Lambda | Execution (30s avg) | $0.10 |
| Bedrock Titan | Query embedding (50 tokens) | $0.000005 |
| Pinecone | Vector search | FREE (free tier) |
| Bedrock Claude | Generation (500 tokens) | $0.38 |
| **Total** | | **~$0.48** |

### Monthly Estimates:
| Usage Level | Queries/Month | Total Cost |
|-------------|---------------|------------|
| Light (demo) | 100 | $0.05 |
| Medium | 1,000 | $0.48 |
| Production | 10,000 | $4.80 |
| Enterprise | 100,000 | $48.00 |

### One-Time Setup Costs:
- Process 1000 Kaggle records: ~$0.10 (Titan embeddings)
- Pinecone setup: FREE (free tier)
- Total: **< $0.20**

---

## Security Architecture

```
┌─────────────────────────────────────────────┐
│          Security Layers                     │
├─────────────────────────────────────────────┤
│ 1. Transport: WSS (TLS 1.2+)                │
│ 2. Authentication: API Key validation       │
│ 3. Rate Limiting: 10 req/min per connection │
│ 4. IAM: Least privilege Lambda role         │
│ 5. Pinecone: API key in Lambda env vars     │
│ 6. Bedrock: Model access controls           │
└─────────────────────────────────────────────┘
```

---

## Scalability Considerations

### Current Limits:
- **Lambda**: 10 concurrent executions (configurable)
- **API Gateway**: 10,000 concurrent WebSocket connections
- **Pinecone Free**: 100K vectors, 100K queries/month
- **Bedrock**: Regional quotas (varies by model)

### Production Scaling:
```
┌─────────────────────────────────────────────────┐
│  Add for Production:                            │
│  • DynamoDB: Connection state persistence       │
│  • CloudFront: Global CDN for frontend          │
│  • Pinecone Standard: Unlimited operations      │
│  • CloudWatch: Enhanced monitoring & alerts     │
│  • AWS WAF: DDoS protection                     │
│  • Cognito: User authentication                 │
└─────────────────────────────────────────────────┘
```

---

## Deployment Architecture

```
Development (Local)          AWS Development           Production
┌─────────────────┐         ┌─────────────────┐       ┌─────────────────┐
│ npm run dev     │         │ CloudFormation  │       │ CloudFormation  │
│ localhost:5173  │   →     │ Stack (dev)     │  →    │ Stack (prod)    │
│                 │         │ • Lambda        │       │ • Lambda        │
│ Connects to:    │         │ • API Gateway   │       │ • API Gateway   │
│ • AWS Bedrock   │         │ • IAM Roles     │       │ • WAF           │
│ • Pinecone      │         └─────────────────┘       │ • CloudFront    │
└─────────────────┘                                   └─────────────────┘
```

---

## Monitoring & Observability

```
┌────────────────────────────────────────────────────┐
│              CloudWatch Dashboards                  │
├────────────────────────────────────────────────────┤
│ • Lambda invocations & errors                      │
│ • Bedrock API latency                              │
│ • WebSocket connection count                       │
│ • Rate limit violations                            │
│ • Pinecone query performance (external)            │
│ • End-to-end response time                         │
└────────────────────────────────────────────────────┘
```

---

## Technology Stack Summary

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Frontend** | React + TypeScript + Vite | User interface |
| **API** | API Gateway WebSocket | Real-time communication |
| **Compute** | AWS Lambda (Python 3.9) | Business logic & orchestration |
| **Embeddings** | Bedrock Titan Embeddings | Text → Vector conversion |
| **Vector DB** | Pinecone | Semantic search |
| **LLM** | Bedrock Claude 3 Haiku | Answer generation |
| **IaC** | CloudFormation | Infrastructure deployment |
| **Monitoring** | CloudWatch | Logs & metrics |
| **Data Source** | Kaggle Datasets | Training knowledge base |

