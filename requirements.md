# The Claim Engine - Requirements Document

## Project Overview
The Claim Engine is a voice-activated, multilingual AI assistant that simplifies the discovery and claiming of government benefits for rural users. It bridges the gap between complex government portals and end-users by automating eligibility assessment and claim submission.

## Problem Statement
Rural users struggle to access government benefits due to:
- Complex application portals (DBT, State portals) with unintuitive interfaces
- Language barriers in non-English speaking regions
- Lack of awareness about available schemes and subsidies
- High dropout rates during the application process
- Manual form-filling requirements that require technical literacy

## Solution Overview
The Claim Engine provides an end-to-end empowerment solution that:
1. Identifies user eligibility through conversational AI
2. Guides users through the "Claim Path" with step-by-step instructions and pre-filled data
3. Reduces friction by preparing all required information upfront and providing direct links to application pages
4. Works *with* government systems (via APIs) rather than automating against them

## Functional Requirements

### 1. Speech & Language Interface
- **Multilingual Voice Input**: Accept voice input in multiple Indian languages
- **Automatic Language Detection**: Identify user's language automatically
- **Speech-to-Text Conversion**: Use Amazon Transcribe for real-time audio processing
- **Text-to-Speech Output**: Use AWS Polly to provide responses in user's native language with natural tone and accent control

### 2. User Profile & Eligibility Assessment
- **Voice-Based Profile Collection**: Users describe their profile (e.g., "I am a farmer with 2 acres of land")
- **Data Extraction**: Extract structured data (occupation, land size, income, family details) from conversational input
- **Document Support**: Accept uploaded documents (Aadhaar, Ration Card) for data extraction using Amazon Textract
- **Eligibility Matching**: Match user profile against government scheme database to identify applicable benefits

### 3. Scheme Discovery & Matching
- **Comprehensive Scheme Database**: Maintain database of government schemes, subsidies, and benefits
- **Intelligent Search**: Use Amazon Kendra for semantic search across scheme eligibility criteria
- **Ranking & Prioritization**: Rank schemes by relevance and benefit amount
- **Detailed Scheme Information**: Provide scheme details, eligibility criteria, and application requirements

### 4. Guided Portal Navigation & Data Preparation
- **Deep Linking**: Generate direct links to specific scheme login/registration pages
- **Smart Form Structure Analysis**: Use headless browser to scrape form structure and identify required fields (read-only, no submission)
- **Intelligent Data Extraction**: Use LLM to extract relevant data from user's uploaded documents (Aadhaar, Ration Card, etc.)
- **Form Field Mapping**: Intelligently map extracted user data to portal form fields
- **Pre-filled PDF Generation**: Generate downloadable PDFs with user data pre-filled for offline submission
- **Smart Form Preview**: Display form preview showing how data will appear on actual portal
- **Copy-Paste Assistance**: Provide field-by-field guidance with copy-paste ready values for manual entry
- **Portal Integration (Official)**: Work with government APIs (API Setu) for direct submission where available
- **User-Controlled Submission**: User retains full control—system guides but never submits without explicit user action

### 5. Document & Data Integration
- **API Setu Integration**: Connect to API Setu (Digital India) to fetch verified user documents automatically
- **OCR Capabilities**: Extract data from uploaded documents using Amazon Textract
- **Data Validation**: Verify extracted data against government records
- **Secure Data Handling**: Encrypt and securely store user information

### 6. Claim Submission & Tracking
- **API-Based Submission**: Submit claims directly via government APIs (API Setu) where available
- **Guided Manual Submission**: For portals without APIs, provide step-by-step guidance with pre-filled data
- **Confirmation Handling**: Capture and process confirmation emails/OTPs
- **Status Tracking**: Provide real-time claim status updates via API polling or webhooks
- **Notification System**: Send SMS/Email notifications via Amazon SNS

## Non-Functional Requirements

### Performance
- Voice processing latency: < 2 seconds for transcription
- Eligibility matching: < 5 seconds for scheme recommendations
- Portal navigation: < 10 seconds per form submission

### Scalability
- Support concurrent users through AWS Lambda auto-scaling
- Handle peak loads during government benefit announcement periods
- Serverless architecture for cost efficiency

### Security
- End-to-end encryption for user data
- Compliance with Indian data protection regulations
- Secure API authentication with government systems
- PII data masking in logs

### Reliability
- 99.5% uptime SLA
- Automatic retry mechanisms for failed submissions
- Fallback options for portal navigation failures
- Data backup and disaster recovery

### Accessibility
- Support for low-bandwidth environments
- Offline capability for basic features
- Voice-first interface for users with limited literacy
- Support for feature phones and basic smartphones

## Technical Stack

### Frontend
- Voice interface with multilingual support
- Web/Mobile application for user interaction
- Real-time status dashboard

### Backend Services
- AWS Lambda for serverless compute
- Amazon Bedrock for LLM-based reasoning
- Amazon Transcribe for speech-to-text
- AWS Polly for text-to-speech
- Amazon Kendra for intelligent search
- Amazon Textract for document processing

### Data Layer
- Amazon DynamoDB for user profiles and claim tracking
- Amazon Bedrock for scheme database and eligibility rules
- API Setu integration for government data

### Execution Layer
- AWS Lambda for orchestration
- Headless Browser (Playwright/Puppeteer): Read-only form structure analysis (no submission)
- Amazon Bedrock (LLM): Intelligent data extraction from documents
- API Setu integration for direct government submissions
- Amazon SES for email notifications
- Amazon SNS for SMS notifications
- PDF generation service for pre-filled forms

## Success Metrics
- User adoption rate among rural population
- Claim success rate (% of submitted claims approved)
- Average time to claim submission
- User satisfaction score
- Cost per successful claim
- Language coverage and accuracy

## Constraints & Assumptions
- **Legal Constraint**: No automated form submission on government portals without explicit API support or legal authorization
- **Technical Constraint**: Government sites use CAPTCHAs and rate limiting—design prioritizes API-first approach
- Government portal APIs may have limited availability—fallback to guided manual submission
- Portal structures may change—minimize maintenance burden by using official APIs
- User internet connectivity may be intermittent
- Compliance with government data sharing policies and Terms of Service required
- Initial focus on schemes with API Setu support or official integration channels
- User consent required for all data sharing with government systems

## Timeline & Milestones
- Phase 1: Core voice interface and eligibility matching (MVP)
- Phase 2: Portal automation and API integration
- Phase 3: Multi-language expansion and optimization
- Phase 4: Scale and government partnership integration
