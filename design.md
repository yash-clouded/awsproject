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
- Headless Browser (Playwright/Puppeteer): Read-only form structure analysis
- Amazon Bedrock (LLM): Intelligent data extraction from documents
- API Setu: Direct government API integration for claim submission
- Amazon Textract: Document processing
- Amazon SES/SNS: Notifications
- PDF Generation Service: Pre-filled form generation

**Portal Navigation Flow (Hybrid Approach):**
```
Scheme Selected
    ↓
Step 1: Analyze Portal Form Structure
    ├─ Use headless browser to scrape form fields (read-only)
    ├─ Identify required fields, field types, validation rules
    ├─ Extract form structure without submitting anything
    └─ Store form schema in cache
    ↓
Step 2: Extract User Data from Documents
    ├─ User uploads documents (Aadhaar, Ration Card, etc.)
    ├─ Use Amazon Textract for OCR
    ├─ Use Amazon Bedrock LLM to intelligently extract relevant data
    ├─ Map extracted data to form fields
    └─ Validate extracted data
    ↓
Step 3: Check Submission Method
    ├─ API Available? → Use API Setu for direct submission
    └─ No API? → Continue to Step 4
    ↓
Step 4: Generate Pre-filled Submission Package
    ├─ Create pre-filled PDF with user data
    ├─ Generate form preview showing how data appears on portal
    ├─ Create copy-paste assistance guide (field-by-field values)
    ├─ Provide deep link to portal
    └─ Send package to user
    ↓
Step 5: User Submission
    ├─ User logs into portal
    ├─ User fills form using pre-filled PDF or copy-paste guide
    ├─ User submits on portal (user retains full control)
    └─ User shares confirmation with system
    ↓
Step 6: Confirmation & Tracking
    ├─ System captures confirmation details
    ├─ Store claim record in DynamoDB
    └─ Send confirmation to user
```

**Key Capabilities:**
- Read-only form structure analysis (no submission, no ToS violation)
- Intelligent LLM-based data extraction from documents
- Pre-filled PDF generation for offline submission
- Form preview showing exact portal appearance
- Copy-paste assistance for manual field entry
- API integration with government systems (API Setu)
- User retains full control over submission
- Minimal maintenance burden (form structure cached, not hardcoded)

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

### Workflow 3: Guided Claim Submission (Hybrid Approach)

```
1. Lambda receives submission request
2. Step 1: Analyze Portal Form Structure
   a. Use headless browser to scrape form fields (read-only)
   b. Identify required fields and validation rules
   c. Cache form schema
3. Step 2: Extract User Data
   a. User uploads documents (Aadhaar, Ration Card, etc.)
   b. Use Textract for OCR
   c. Use Bedrock LLM to intelligently extract relevant data
   d. Map data to form fields
4. Step 3: Check Submission Method
   a. If API available: Call API Setu with user consent
   b. If no API: Continue to Step 4
5. Step 4: Generate Pre-filled Package
   a. Create pre-filled PDF with user data
   b. Generate form preview
   c. Create copy-paste assistance guide
   d. Send to user with deep link
6. Step 5: User Submission
   a. User logs into portal
   b. User fills form using pre-filled PDF or copy-paste guide
   c. User submits on portal
   d. User shares confirmation with system
7. Step 6: Confirmation & Tracking
   a. System captures confirmation details
   b. Store claim record in DynamoDB
   c. Send confirmation to user
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
│  - API Setu Integration                 │
│  - Notification Service                 │
└────────────────┬────────────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼──┐  ┌─────▼──┐  ┌──────▼──┐
│DynamoDB│ │  S3   │  │ Kendra  │
└────────┘  └───────┘  └─────────┘
```

## Legal & Technical Risk Mitigation

### Why We Use a Hybrid Approach (Not Full Automation)

**Technical Risks of Full Automation:**
- Government portals use CAPTCHAs, rate limiting, and bot detection
- Portal HTML structures change frequently, breaking automation
- Maintenance burden grows exponentially with each new portal
- Session management and authentication are complex and fragile

**Legal Risks of Full Automation:**
- Terms of Service violations on government websites
- Potential violation of Computer Fraud and Abuse Act (CFAA) equivalent in India
- Liability if automated submissions cause data corruption or duplicate claims
- Regulatory scrutiny from government agencies

**Business Risks of Full Automation:**
- Portal operators can block automation at any time
- Reputational damage if automation causes claim rejections
- Compliance violations could result in service shutdown

### Our Hybrid Solution: Read-Only Analysis + LLM Extraction + User Control

**Why This Works:**

1. **Read-Only Form Structure Analysis**
   - Use headless browser only to READ form structure (what fields exist)
   - Never submit anything
   - No Terms of Service violation (similar to browser inspection tools)
   - Form structure cached, not hardcoded—minimal maintenance
   - Compliant with portal policies

2. **Intelligent LLM-Based Data Extraction**
   - Use Amazon Bedrock to extract relevant data from user documents
   - No manual data entry required from users
   - Accurate field mapping based on form structure
   - Reduces user friction significantly

3. **Pre-filled PDF + Copy-Paste Assistance**
   - Generate downloadable PDF with all data pre-filled
   - Provide copy-paste ready values for each form field
   - User sees exactly how data will appear on portal
   - User retains full control over submission

4. **API Setu Integration (When Available)**
   - Direct government API submission where available
   - Eliminates all portal interaction risks
   - Government-approved approach
   - Scalable and maintainable

### Compliance Strategy
- Explicit user consent for all data sharing
- Clear disclosure of what system can and cannot do
- Audit trail of all submissions
- Regular legal review of API usage
- Transparent communication with government partners
- Read-only browser access compliant with portal policies

## Error Handling & Resilience

### Failure Scenarios
1. **API Unavailability**: Fall back to guided manual submission with pre-filled data
2. **Portal Changes**: Minimal impact since we don't scrape or automate portals
3. **API Setu Timeout**: Retry with exponential backoff, notify user to submit manually
4. **Network Interruption**: Queue request, retry when connection restored
5. **User Submission Failure**: Provide troubleshooting guide and alternative submission methods

### Recovery Mechanisms
- **Dead Letter Queues**: Capture failed API submissions for manual review
- **Circuit Breaker Pattern**: Prevent cascading failures in API calls
- **Graceful Degradation**: Always provide manual submission as fallback
- **User Support Queue**: Route complex cases to human support

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
