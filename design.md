# The Claim Engine - Design Document

## Architecture Overview

The Claim Engine follows a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                    USER INTERFACE LAYER                      │
│              (Voice + Web/Mobile Frontend)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                  ORCHESTRATION LAYER                         │
│         (AWS Lambda - Request Routing & State Mgmt)          │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┬──────────────┐
        │            │            │              │
┌───────▼──┐  ┌──────▼──┐  ┌─────▼──┐  ┌──────▼──┐
│ SPEECH & │  │REASONING│  │MATCHING│  │EXECUTION│
│ LANGUAGE │  │  LAYER  │  │ LAYER  │  │ LAYER   │
│  LAYER   │  │         │  │        │  │         │
└──────────┘  └─────────┘  └────────┘  └─────────┘
     │            │            │            │
     │            │            │            │
┌────▼────────────▼────────────▼────────────▼──────┐
│              DATA & INTEGRATION LAYER             │
│  (Databases, APIs, External Services)            │
└──────────────────────────────────────────────────┘
```

## Detailed Component Design

### 1. Speech & Language Layer

**Components:**
- Amazon Transcribe: Converts voice input to text
- Amazon Bedrock (LLM): Processes natural language
- AWS Polly: Converts responses back to speech

**Flow:**
```
User Voice Input
    ↓
Amazon Transcribe (Multilingual)
    ↓
Language Detection & NLU
    ↓
Intent Extraction (Profile, Query, Confirmation)
    ↓
AWS Polly (Text-to-Speech)
    ↓
User Audio Response
```

**Key Features:**
- Real-time transcription with automatic language detection
- Support for Indian languages (Hindi, Tamil, Telugu, Kannada, etc.)
- Natural speech synthesis with regional accents
- Fallback to text interface for low-bandwidth scenarios

### 2. Reasoning Layer (The Brain)

**Components:**
- Amazon Bedrock with Claude/Llama foundation models
- Custom eligibility rules engine
- Conversation state management

**Responsibilities:**
- Extract structured data from conversational input
- Evaluate eligibility against scheme criteria
- Handle complex logic and edge cases
- Maintain conversation context across sessions

**Example Logic Flow:**
```
User Input: "I am a farmer with 2 acres of land in Maharashtra"
    ↓
Extract Entities:
  - Occupation: Farmer
  - Land Size: 2 acres
  - State: Maharashtra
    ↓
Query Eligibility Rules:
  - PM-KISAN: ✓ Eligible (< 2 hectares)
  - PMAY: ✓ Eligible (rural farmer)
  - Crop Insurance: ✓ Eligible
    ↓
Rank by Benefit Amount & Relevance
    ↓
Return Recommendations
```

### 3. Matching Layer (The Knowledge Base)

**Components:**
- Amazon Kendra: Intelligent search over scheme database
- DynamoDB: Scheme metadata and eligibility criteria
- API Setu: Government scheme information

**Data Structure:**
```json
{
  "scheme_id": "PM-KISAN-001",
  "name": "Pradhan Mantri Kisan Samman Nidhi",
  "eligibility_criteria": {
    "occupation": ["farmer"],
    "land_size_max": 2,
    "income_max": 500000,
    "states": ["all"]
  },
  "benefit_amount": 6000,
  "application_url": "https://pmkisan.gov.in",
  "documents_required": ["aadhaar", "land_records"],
  "processing_time": "30-45 days"
}
```

**Search Capabilities:**
- Semantic search across scheme descriptions
- Faceted search by state, occupation, benefit type
- Ranking by relevance and benefit amount

### 4. Execution Layer (The Muscle)

**Components:**
- AWS Lambda: Orchestration and backend logic
- Playwright/Puppeteer: Headless browser automation
- Amazon Textract: Document processing
- Amazon SES/SNS: Notifications

**Portal Navigation Flow:**
```
Scheme Selected
    ↓
Generate Deep Link to Portal
    ↓
Launch Headless Browser
    ↓
Navigate to Application Page
    ↓
Extract Form Fields
    ↓
Map User Data to Form Fields
    ↓
Auto-fill Forms
    ↓
Handle OTP/Verification
    ↓
Submit Application
    ↓
Capture Confirmation
    ↓
Send Notification to User
```

**Key Capabilities:**
- Intelligent form field detection
- Handling of dynamic forms and JavaScript-heavy portals
- OTP interception and auto-fill
- Screenshot capture for audit trail
- Error recovery and retry logic

### 5. Data & Integration Layer

**Databases:**
- **DynamoDB**: User profiles, claim history, session state
- **S3**: Document storage, audit logs
- **RDS**: Scheme database (optional, for complex queries)

**External Integrations:**
- **API Setu**: Fetch verified Aadhaar, PAN, bank details
- **Government Portals**: DBT, State benefit portals
- **Email/SMS**: Amazon SES, SNS for notifications

**Data Flow:**
```
User Data Collection
    ↓
Encryption & Storage (DynamoDB)
    ↓
API Setu Integration (Fetch Verified Data)
    ↓
Eligibility Matching
    ↓
Portal Submission
    ↓
Status Tracking & Notifications
```

## System Workflows

### Workflow 1: User Onboarding & Profile Creation

```
1. User initiates voice call/app
2. System greets in detected language
3. User describes profile conversationally
4. System extracts and confirms details
5. Optional: User uploads documents (Aadhaar, Ration Card)
6. System processes documents via Textract
7. Profile stored in DynamoDB
8. Ready for eligibility matching
```

### Workflow 2: Eligibility Assessment & Scheme Discovery

```
1. System queries Kendra with user profile
2. Bedrock evaluates eligibility rules
3. Schemes ranked by relevance
4. System presents top 5 schemes to user
5. User selects scheme of interest
6. System provides detailed scheme information
7. User confirms intent to apply
```

### Workflow 3: Automated Claim Submission

```
1. Lambda receives submission request
2. Fetch scheme portal URL and form structure
3. Launch headless browser
4. Navigate to application page
5. Extract form fields
6. Map user data to fields
7. Auto-fill forms with validation
8. Handle authentication (OTP, etc.)
9. Submit application
10. Capture confirmation details
11. Store claim record in DynamoDB
12. Send confirmation to user via SMS/Email
13. Schedule status tracking
```

### Workflow 4: Claim Status Tracking

```
1. System periodically checks claim status
2. Query government portal or API
3. Update claim status in DynamoDB
4. If status changed, notify user
5. Provide next steps if action needed
6. Archive completed claims
```

## API Design

### Core APIs

**1. Voice Input Processing**
```
POST /api/v1/voice/process
{
  "audio_stream": "base64_encoded_audio",
  "language": "auto_detect",
  "session_id": "user_session_123"
}
Response:
{
  "transcribed_text": "I am a farmer with 2 acres",
  "intent": "profile_update",
  "entities": {
    "occupation": "farmer",
    "land_size": "2 acres"
  }
}
```

**2. Eligibility Check**
```
POST /api/v1/eligibility/check
{
  "user_profile": {
    "occupation": "farmer",
    "land_size": 2,
    "state": "Maharashtra"
  }
}
Response:
{
  "eligible_schemes": [
    {
      "scheme_id": "PM-KISAN",
      "name": "Pradhan Mantri Kisan Samman Nidhi",
      "match_score": 0.95,
      "benefit_amount": 6000
    }
  ]
}
```

**3. Claim Submission**
```
POST /api/v1/claims/submit
{
  "user_id": "user_123",
  "scheme_id": "PM-KISAN",
  "user_data": { ... }
}
Response:
{
  "claim_id": "CLAIM-2024-001",
  "status": "submitted",
  "confirmation_number": "ABC123XYZ",
  "estimated_processing_time": "30-45 days"
}
```

**4. Claim Status**
```
GET /api/v1/claims/{claim_id}/status
Response:
{
  "claim_id": "CLAIM-2024-001",
  "status": "under_review",
  "last_updated": "2024-02-08T10:30:00Z",
  "next_steps": "Awaiting document verification"
}
```

## Security & Privacy Design

### Data Protection
- **Encryption at Rest**: AES-256 for DynamoDB and S3
- **Encryption in Transit**: TLS 1.3 for all API communications
- **PII Masking**: Sensitive data masked in logs and audit trails

### Authentication & Authorization
- **User Authentication**: OTP-based verification via SMS
- **API Authentication**: AWS IAM roles and API keys
- **Session Management**: Secure session tokens with expiration

### Compliance
- **Data Residency**: Data stored in India (AWS Mumbai region)
- **GDPR/Privacy**: User consent for data collection and processing
- **Government Compliance**: Adherence to Digital India standards

### Audit & Monitoring
- **Audit Logs**: All user actions logged with timestamps
- **CloudWatch Monitoring**: Real-time system health monitoring
- **Anomaly Detection**: Alert on suspicious activities

## Scalability & Performance

### Horizontal Scaling
- **Lambda Auto-scaling**: Automatic scaling based on concurrent requests
- **DynamoDB On-demand**: Pay-per-request pricing for variable loads
- **CloudFront CDN**: Cache static assets and API responses

### Performance Optimization
- **Caching Strategy**: Cache scheme database in memory
- **Batch Processing**: Batch claim status checks
- **Async Processing**: Use SQS for long-running tasks

### Cost Optimization
- **Serverless Architecture**: Pay only for compute used
- **AWS Free Tier**: Leverage free tier for development
- **Reserved Capacity**: For predictable baseline load

## Deployment Architecture

```
┌─────────────────────────────────────────┐
│         CloudFront (CDN)                │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│      API Gateway (REST/WebSocket)       │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│    Lambda Functions (Microservices)     │
│  - Voice Processing                     │
│  - Eligibility Engine                   │
│  - Portal Automation                    │
│  - Notification Service                 │
└────────────────┬────────────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼──┐  ┌─────▼──┐  ┌──────▼──┐
│DynamoDB│ │  S3   │  │ Kendra  │
└────────┘  └───────┘  └─────────┘
```

## Error Handling & Resilience

### Failure Scenarios
1. **Portal Unavailability**: Retry with exponential backoff, notify user
2. **Form Structure Changes**: Alert admin, fallback to manual submission
3. **API Setu Timeout**: Use cached data, retry asynchronously
4. **Network Interruption**: Queue request, retry when connection restored

### Recovery Mechanisms
- **Dead Letter Queues**: Capture failed submissions for manual review
- **Circuit Breaker Pattern**: Prevent cascading failures
- **Graceful Degradation**: Provide alternative paths when services fail

## Monitoring & Observability

### Key Metrics
- Voice processing latency (p50, p95, p99)
- Eligibility matching accuracy
- Claim submission success rate
- Portal automation success rate
- User satisfaction score

### Logging & Tracing
- **CloudWatch Logs**: Centralized logging for all services
- **X-Ray Tracing**: Distributed tracing for request flows
- **Custom Metrics**: Business metrics via CloudWatch

### Alerting
- High error rates (> 5%)
- Latency spikes (> 10s)
- Failed claim submissions
- API quota exhaustion
