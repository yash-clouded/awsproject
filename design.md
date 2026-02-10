# The Claim Engine - Design Document

## Architecture Overview

The Claim Engine follows a layered architecture with three core execution layers aligned with the hackathon requirements:

```
┌─────────────────────────────────────────────────────────────┐
│                    USER INTERFACE LAYER                      │
│         (Voice + Web/Mobile Frontend - Multilingual)         │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│              ORCHESTRATION LAYER                             │
│         (AWS Lambda - Request Routing & State Mgmt)          │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┬──────────────┐
        │            │            │              │
┌───────▼──┐  ┌──────▼──┐  ┌─────▼──┐  ┌──────▼──┐
│ SPEECH & │  │AI       │  │MATCHING│  │EXECUTION│
│ LANGUAGE │  │REASONING│  │ LAYER  │  │ LAYER   │
│  LAYER   │  │ LAYER   │  │        │  │         │
└──────────┘  └─────────┘  └────────┘  └─────────┘
     │            │            │            │
     │            │            │            │
┌────▼────────────▼────────────▼────────────▼──────┐
│              DATA & INTEGRATION LAYER             │
│  (Databases, APIs, External Services)            │
└──────────────────────────────────────────────────┘
```

## Three Core Execution Layers (Hackathon Focus)

### 1. Speech & Language Layer (Human-to-Machine Translation)

**Purpose**: Enable voice-first interaction in native languages

**Components:**
- Amazon Transcribe: Converts voice input to text (multilingual, automatic language detection)
- Amazon Bedrock (LLM): Processes natural language and extracts intent
- AWS Polly: Converts responses back to speech with natural tone and regional accents

**Capabilities:**
- Real-time multilingual voice processing
- Automatic language detection and switching
- Natural speech synthesis with SSML support for tone/pitch/accent control
- Support for Indian languages (Hindi, Tamil, Telugu, Kannada, Marathi, etc.)
- Fallback to text interface for low-bandwidth scenarios
- **Converts citizen speech ↔ AI responses in native languages**: Seamless bidirectional communication

**Flow:**
```
User Voice Input (Native Language)
    ↓
Amazon Transcribe (Auto Language Detection)
    ↓
Amazon Bedrock LLM (Intent & Entity Extraction)
    ↓
Process Request
    ↓
AWS Polly (Text-to-Speech in Native Language)
    ↓
User Audio Response
```

### 2. AI Reasoning Layer (The Brain)

**Purpose**: Evaluate eligibility and match users with appropriate schemes

**Components:**
- Amazon Bedrock with Claude/Llama foundation models
- Custom eligibility rules engine
- Conversation state management

**Responsibilities:**
- Extract structured data from conversational input
- Evaluate eligibility against scheme criteria using complex logic
- Handle "fuzzy" logic for nuanced eligibility decisions
- Provide explanations for eligibility results
- Maintain conversation context across sessions
- **Amazon Bedrock evaluates user eligibility**: Matches citizens with correct government schemes
- **Provides explanation and next steps**: Clear reasoning for eligibility decisions

**Example Logic:**
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
Return Recommendations with Explanations
```

### 3. Automation & Execution Layer (The Muscle)

**Purpose**: Automate form-filling and claim submission with pre-fill guidance

**Components:**
- AWS Lambda: Orchestration and backend logic
- Headless Browser (Playwright/Puppeteer): Read-only form structure analysis
- Amazon Bedrock (LLM): Intelligent data extraction from documents
- API Setu: Direct government API integration
- Amazon Textract: Document OCR and processing
- Amazon SES/SNS: Notifications and confirmations
- PDF Generation Service: Pre-filled form generation

**Capabilities:**
- Read-only form structure analysis (no submission, compliant)
- Intelligent LLM-based data extraction from documents
- Pre-filled PDF generation for offline submission
- Form preview showing exact portal appearance
- Copy-paste assistance for manual field entry
- API integration with government systems
- Secure claim submission and status tracking
- OTP handling and user verification
- **AWS Lambda automates form filling & portal navigation**: Backend automatically extracts documents, fills online forms, and handles claim processing using serverless functions
- **Secure claim submission and status tracking**: Claims submitted securely with real-time status updates
- **OTP Verification**: Secure OTP confirmation before final submission

**Flow:**
```
Scheme Selected
    ↓
Step 1: Analyze Portal Form Structure
    ├─ Use headless browser to scrape form fields (read-only)
    ├─ Identify required fields and validation rules
    └─ Cache form schema
    ↓
Step 2: Extract User Data from Documents
    ├─ User uploads documents (Aadhaar, Ration Card, etc.)
    ├─ Use Amazon Textract for OCR
    ├─ Use Amazon Bedrock LLM to intelligently extract relevant data
    └─ Map data to form fields
    ↓
Step 3: Check Submission Method
    ├─ API Available? → Use API Setu for direct submission
    └─ No API? → Continue to Step 4
    ↓
Step 4: Generate Pre-filled Submission Package
    ├─ Create pre-filled PDF with user data
    ├─ Generate form preview
    ├─ Create copy-paste assistance guide
    └─ Send to user with deep link
    ↓
Step 5: User Submission
    ├─ User logs into portal
    ├─ User fills form using pre-filled PDF or copy-paste guide
    ├─ User submits on portal
    └─ User shares confirmation with system
    ↓
Step 6: Confirmation & Tracking
    ├─ System captures confirmation details
    ├─ Store claim record in DynamoDB
    └─ Send confirmation to user
```
```

### 4. Data & Integration Layer

**Databases:**
- **DynamoDB**: User profiles, claim history, session state
- **S3**: Document storage, audit logs
- **RDS**: Scheme database (optional, for complex queries)

**External Integrations:**
- **API Setu**: Fetch verified Aadhaar, PAN, bank details automatically
- **Government Portals**: DBT, State benefit portals
- **Email/SMS**: Amazon SES, SNS for notifications and confirmations

**Data Flow:**
```
User Data Collection (Voice/Upload)
    ↓
Encryption & Storage (DynamoDB)
    ↓
API Setu Integration (Fetch Verified Data)
    ↓
Eligibility Matching
    ↓
Portal Submission (API or Guided)
    ↓
Status Tracking & Notifications
```

## System Workflows

### Workflow 1: User Onboarding & Profile Creation

```
1. User initiates voice call/app
2. System greets in detected language
3. User describes profile conversationally (e.g., "I am a farmer with 2 acres")
4. System extracts and confirms details
5. Optional: User uploads documents (Aadhaar, Ration Card)
6. System processes documents via Textract + Bedrock LLM
7. Profile stored in DynamoDB with encryption
8. Ready for eligibility matching
```

### Workflow 2: Eligibility Assessment & Scheme Discovery (Days → Minutes)

```
1. System queries Kendra with user profile
2. Bedrock evaluates eligibility rules against scheme criteria
3. Schemes ranked by relevance and benefit amount
4. System presents top 5 schemes to user in native language
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
   c. Send confirmation to user via SMS/Email
```

### Workflow 4: Claim Status Tracking

```
1. System periodically checks claim status
2. Query government portal or API
3. Update claim status in DynamoDB
4. If status changed, notify user in native language
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

## Hackathon MVP Scope

For the hackathon demo, the system will support the **Beti Bachao Beti Padhao (BBBP) scheme** with live claim navigation:

**MVP Features:**
- Voice-based user profile collection in 2-3 Indian languages
- Eligibility assessment using Bedrock LLM
- Scheme discovery and ranking
- Form structure analysis for BBBP scheme
- Pre-filled PDF generation with user data
- Copy-paste assistance guide
- Live portal navigation demonstration
- Claim confirmation and tracking
- OTP verification workflow

**Out of Scope for MVP:**
- Full multi-language support (focus on 2-3 languages)
- Automated API Setu integration (manual API calls demonstrated)
- Production-grade security (demo-level encryption)
- Multi-scheme support (focus on BBBP scheme)

## Cost & Scalability

**Development Costs:**
- Minimal, leveraged through AWS Credits
- Ideal for hackathon prototyping and initial deployment

**Operational Costs:**
- Scalable pay-as-you-go model with AWS Serverless technologies
- Drastically reduces financial burden for government adoption
- Cost per claim submission: Minimal (Lambda + API calls only)

**Long-Term Viability:**
- Designed for low ongoing costs
- Sustainable solution for wide-scale rural inclusion
- Government-friendly pricing model

## AI Claim Assistant - User Journey

**Key Features:**
- **Voice-first interaction in local languages**: Users interact entirely through voice in their native language
- **Instant scheme eligibility detection**: System immediately identifies applicable schemes based on user profile
- **Personalized scheme recommendations**: Top schemes ranked by relevance and benefit amount
- **One-tap application guidance**: Simple, step-by-step instructions for claim submission
- **Automatic form filling from documents**: Backend extracts data from uploaded documents and pre-fills forms
- **Secure OTP verification before submission**: User confirms details before final submission
- **Real-time application status tracking**: Users receive live updates on claim progress
- **Simple interface designed for rural users**: Minimal text, maximum voice, intuitive navigation

**User Journey Flow:**
1. User calls or opens app
2. System greets in detected language
3. User describes profile (e.g., "I am a farmer with 2 acres")
4. System identifies eligible schemes instantly
5. User selects scheme of interest
6. System provides detailed guidance
7. User uploads documents (Aadhaar, Ration Card)
8. System extracts data and pre-fills forms
9. User reviews and confirms details
10. OTP verification sent
11. Claim submitted securely
12. Real-time status tracking begins
