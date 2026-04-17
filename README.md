# ArterisChatbot

## 1. Project Title & Description

**ArterisChatbot** is a production-grade, serverless AI assistant designed to provide accurate, explainable, and reliable answers about Arteris, its semiconductor System IP portfolio, and Network-on-Chip (NoC) technologies.

The system combines:
- Deterministic knowledge retrieval for predefined UI questions
- Hybrid RAG (Retrieval-Augmented Generation) for free-text queries
- General LLM capabilities for non-Arteris questions
- Conversational memory for contextual interactions

It is fully implemented using AWS serverless services with no vector database or OpenSearch.

---

## 2. Purpose and Business Objective

The Arteris Assistant is designed to:

- Support **pre-sales conversations**
- Help users understand **System IP and NoC architectures**
- Provide **accurate answers grounded in Arteris knowledge base**
- Enable **deterministic answers for UI-driven questions**
- Reduce reliance on human experts via **automated support**
- Provide **explainability, traceability, and observability**

---

## 3. Scope and Supported Models

### Supported capabilities

- Arteris product knowledge (FlexNoC, Ncore, Magillem, Multi-Die)
- Semiconductor education (SoC, NoC, interconnect)
- Conversational Q&A
- Summarization and translation
- Follow-up questions with memory

### Supported response modes

- deterministic_kb
- kb_rag
- general_llm
- clarification

### Models used

- Chat model:
  arn:aws:bedrock:eu-west-1:598967424083:inference-profile/eu.anthropic.claude-sonnet-4-20250514-v1:0

- Embedding model:
  amazon.titan-embed-text-v2:0

---

## 4. Architecture

### 4.1 High-Level Architecture
```
Frontend UI
   |
   v
API Gateway HTTP API
   |
   +--> POST /chat --------> ARTERIS-chat-handler Lambda
   |                            |
   |                            +--> S3: load KB index + optional prompts/config
   |                            +--> DynamoDB: sessions, messages, memory
   |                            +--> Bedrock: Titan embeddings for query
   |                            +--> Bedrock: Claude Sonnet 4 via inference profile
   |                            +--> CloudWatch Logs + EMF metrics
   |
   +--> POST /feedback ----> ARTERIS-feedback-handler Lambda
   |                            |
   |                            +--> DynamoDB: feedback
   |                            +--> CloudWatch Logs
   |
   +--> POST /escalate ----> ARTERIS-escalate-handler Lambda
   |                            |
   |                            +--> DynamoDB: escalations
   |                            +--> CloudWatch Logs
   |
   +--> GET /metrics ------> ARTERIS-metrics-handler Lambda
                                |
                                +--> DynamoDB scans/queries
                                +--> CloudWatch Logs
```
---

### 4.2 Component Description

**API Gateway HTTP API**
- Public HTTPS entry point
- Routes requests to Lambda
- Handles CORS

**ARTERIS-chat-handler**
- Validates request
- Selects response mode
- Loads KB from S3 (cached)
- Executes deterministic or hybrid retrieval
- Builds prompt
- Calls Bedrock
- Stores conversation
- Returns structured response

**S3**
- Stores knowledge base
- Stores optional prompts/config

**DynamoDB**
- Sessions
- Messages
- Memory
- Feedback
- Escalations

**Bedrock**
- Titan embeddings for semantic retrieval
- Claude Sonnet 4 for generation

**CloudWatch**
- Logs
- Metrics
- Debug traces

---

## 5. Technical Flows

### Data Flow

1. User sends request to `/chat`
2. API Gateway routes to Lambda
3. Lambda loads KB + memory
4. Lambda performs retrieval
5. Prompt is constructed
6. Bedrock generates response
7. Response stored + returned

---

### Retrieval Flow
```
User question
   |
   +--> deterministic hint? → exact match
   |
   +--> otherwise:
         |
         +--> Arteris-related → hybrid retrieval
         |
         +--> generic → LLM only
```
---

### Retrieval Scoring Logic

final_score = (0.7 * semantic_score) + (0.3 * keyword_score)

- semantic_score: cosine similarity from embeddings
- keyword_score: token overlap

---

### Conversational Memory Flow
```
Incoming request
   |
   +--> load persistent memory
   +--> load session summary
   +--> load last 6–10 turns
   |
   v
prompt assembly
   |
   v
store new messages
   |
   +--> update summary periodically
```
---

## 6. Functional Flow

### 6.1 User Interaction Flow

1. User sends message
2. System classifies intent:
   - unclear → clarification
   - UI question → deterministic_kb
   - Arteris → kb_rag
   - other → general_llm
3. Retrieval executed
4. Prompt built
5. Response generated
6. Stored and returned

---

### 6.2 Escalation Flow

1. User triggers escalation
2. Request sent to `/escalate`
3. Lambda stores escalation record
4. System returns:

   An expert will join the conversation shortly.

---

### 6.3 Metrics Calculation

Tracked metrics:
- total requests
- active sessions
- feedback counts
- escalation count
- mode usage

Derived metrics:
- KB coverage rate
- escalation rate by topic
- dislike rate by product

---

### 6.4 Observability Design / Flow

Each request logs:
- request_id
- session_id
- user_id
- mode
- latency
- retrieved documents
- knowledge usage

Metrics emitted via EMF:
- total requests
- mode counts
- latency

---

### 6.5 Explainability Design / Flow

Each response includes:
- mode
- used_knowledge
- sources

Optional debug:
- retrieval_path_used
- retrieved_chunks
- scores
- prompt summary

---

## 7. Architecture Overview (Detailed)

### Textual architecture diagram
```
Frontend UI
   |
   v
API Gateway HTTP API
   |
   +--> POST /chat --------> ARTERIS-chat-handler Lambda
   |
   +--> POST /feedback ----> ARTERIS-feedback-handler
   |
   +--> POST /escalate ----> ARTERIS-escalate-handler
   |
   +--> GET /metrics ------> ARTERIS-metrics-handler
```
---

### End-to-End Data Flow

1. Request received
2. KB loaded from S3
3. Memory loaded from DynamoDB
4. Mode selected
5. Retrieval executed
6. Prompt assembled
7. Bedrock called
8. Response stored
9. Response returned

---

### Deterministic Retrieval Flow

- Uses exact metadata filtering
- No embeddings
- No scoring
- Returns single best chunk

---

### Memory Flow

- Recent turns
- Session summary
- Persistent memory

---

### API Interactions

- POST /chat
- POST /feedback
- POST /escalate
- GET /metrics

---

## 8. Resource Naming Plan

S3
- ARTERIS-kb-eu-west-1-598967424083

DynamoDB
- ARTERIS-Sessions
- ARTERIS-Messages
- ARTERIS-Memory
- ARTERIS-Feedback
- ARTERIS-Escalations

IAM
- ARTERIS-Lambda-Shared-Role
- ARTERIS-Lambda-Shared-Policy

Lambda
- ARTERIS-chat-handler
- ARTERIS-feedback-handler
- ARTERIS-escalate-handler
- ARTERIS-metrics-handler

API Gateway
- ARTERIS-http-api

CloudWatch
- /aws/lambda/ARTERIS-*

---

## 9. Implementation Sequence

1. Create S3 bucket
2. Upload KB
3. Create DynamoDB tables
4. Enable TTL
5. Create IAM role + policy
6. Deploy Lambdas
7. Configure environment variables
8. Create API Gateway
9. Create integrations
10. Create routes
11. Deploy stage
12. Add permissions
13. Test endpoints
14. Monitor logs

---

## 10. API Examples

### Chat
```
    curl -X POST "$API_URL/chat" \
      -H "Content-Type: application/json" \
      -d '{
        "session_id": "sess-001",
        "message": "What is FlexNoC?"
      }'
```
---

### Feedback
```
    curl -X POST "$API_URL/feedback" \
      -H "Content-Type: application/json" \
      -d '{
        "session_id": "sess-001",
        "response_id": "resp-001",
        "feedback": "like"
      }'
```
---

### Escalate
```
    curl -X POST "$API_URL/escalate" \
      -H "Content-Type: application/json" \
      -d '{
        "session_id": "sess-001",
        "user_message": "Need help",
        "assistant_response": "..."
      }'
```
---

## 11. Key Design Principles

- No hallucination
- Deterministic answers for UI questions
- KB-first retrieval
- Explainable responses
- Serverless architecture
- No vector database
- Fully observable system

---

## 12. Summary

ArterisChatbot is a **production-ready AI backend** combining:

- Deterministic retrieval
- Hybrid RAG
- Conversational memory
- Observability
- Explainability

All implemented with:
- AWS Lambda
- API Gateway
- S3
- DynamoDB
- Bedrock
- CloudWatch

This architecture ensures:
- accuracy
- scalability
- maintainability
- traceability
