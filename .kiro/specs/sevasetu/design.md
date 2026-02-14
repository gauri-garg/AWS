# Design Document: SevaSetu

## Overview

SevaSetu is a serverless voice assistant application built entirely on AWS free tier services that enables rural Indian citizens to access government scheme information through voice interactions in regional languages. The system initially supports Hindi and Marathi as specified in the requirements, with the architecture designed to support additional Indian languages (Tamil, Telugu, Bengali, Kannada, Malayalam, Gujarati, Punjabi, Odia, and Assamese) in future phases.

The system leverages AWS Lambda for compute, API Gateway for request routing, S3 for document storage, DynamoDB for user history, and AWS AI services for language processing. The architecture is specifically designed to operate within AWS free tier limits while maintaining functionality and user experience.

**Key Design Decision - AI Summarization**: The requirements specify using Amazon Bedrock for AI summarization (Requirement 4). However, Amazon Bedrock is not available in the AWS free tier. To stay within free tier constraints, this design uses **Amazon Comprehend** (50K units/month free) for entity extraction combined with rule-based extractive summarization. This approach provides simplified summaries while remaining cost-free. If budget becomes available, the system can be upgraded to use Amazon Bedrock for higher-quality abstractive summarization.

The system implements aggressive caching, request throttling, and usage monitoring to stay within free tier boundaries. The system processes voice queries through a pipeline: speech-to-text conversion, scheme retrieval, AI summarization, and text-to-speech response generation, all while maintaining language consistency throughout the interaction.

## Requirements Coverage

This design addresses all 12 requirements from the requirements document:

1. **Requirement 1 (Voice Input Processing)**: Handled by Voice Input Handler and Speech-to-Text Lambda using AWS Transcribe
2. **Requirement 2 (Language Detection and Consistency)**: Language detection in Speech-to-Text Lambda, consistency maintained throughout pipeline
3. **Requirement 3 (Scheme Information Retrieval)**: Implemented by Scheme Retriever Lambda accessing S3 bucket
4. **Requirement 4 (AI-Powered Summarization)**: Implemented by AI Summarizer Lambda using Amazon Comprehend (free tier alternative to Bedrock)
5. **Requirement 5 (Voice Response Generation)**: Handled by Text-to-Speech Lambda using AWS Polly
6. **Requirement 6 (User History Management)**: Implemented by History Manager Lambda writing to DynamoDB
7. **Requirement 7 (Serverless Architecture)**: Entire system built on AWS Lambda, API Gateway, S3, and DynamoDB
8. **Requirement 8 (Low Latency Optimization)**: Aggressive caching strategy, audio compression, and optimized Lambda configurations
9. **Requirement 9 (Security and Authentication)**: API Gateway authentication, TLS encryption, IAM roles, data encryption at rest
10. **Requirement 10 (High Availability and Reliability)**: Multi-AZ Lambda deployment, DynamoDB replication, retry logic
11. **Requirement 11 (Scalability)**: Auto-scaling Lambda, on-demand DynamoDB, throttling mechanisms
12. **Requirement 12 (Error Handling and User Feedback)**: Comprehensive error handling with language-specific messages, CloudWatch logging

## Architecture

### High-Level Architecture

The SevaSetu system follows a serverless microservices architecture with aggressive caching to stay within AWS free tier limits:

```
[User Device] 
    ↓ (Voice Input - HTTP/HTTPS)
[API Gateway] (1M calls/month free)
    ↓ (Authenticated Request)
[Lambda: Request Orchestrator] (400K GB-sec/month free)
    ↓
    ├→ [Lambda: Cache Manager] → [DynamoDB: Cache Store] (25GB free)
    ↓     (Check transcription/query/summary cache)
    ↓
    ├→ [Lambda: Speech-to-Text] → [AWS Transcribe] (60 min/month free)
    ↓     (Only if cache miss)
    ↓
    ├→ [Lambda: Scheme Retriever] → [S3: Scheme Documents] (5GB, 20K GET/month free)
    ↓     (Only if cache miss)
    ↓
    ├→ [Lambda: AI Summarizer] → [Amazon Comprehend] (50K units/month free)
    ↓                          → [AWS Translate] (2M chars/month free)
    ↓     (Only if cache miss)
    ↓
    ├→ [Lambda: Text-to-Speech] → [AWS Polly] (5M chars/month free)
    ↓
    └→ [Lambda: History Manager] → [DynamoDB: User History] (25GB free)
    ↓
[API Gateway]
    ↓ (Voice Response)
[User Device]
```

**Free Tier Optimization Flow**:
1. Every request checks cache first (DynamoDB - free)
2. Cache hit: Skip expensive services (Transcribe, Comprehend, Translate)
3. Cache miss: Use AWS services, then cache result
4. Target 80%+ cache hit rate to stay within limits

### Architecture Layers

1. **API Layer**: API Gateway handles request routing, authentication, throttling, and response delivery
2. **Orchestration Layer**: Main Lambda function coordinates the workflow across specialized Lambda functions
3. **Processing Layer**: Specialized Lambda functions for speech-to-text, retrieval, summarization, text-to-speech
4. **Data Layer**: S3 for scheme documents, DynamoDB for user history
5. **AI Layer**: AWS Translate for language translation, AWS Transcribe for STT, AWS Polly for TTS, Amazon Comprehend for text analysis

### AWS Free Tier Services and Limits

All services are selected to operate within AWS free tier limits:

- **AWS Lambda**: 
  - 1M free requests per month
  - 400,000 GB-seconds of compute time per month
  - Always free (not just 12 months)

- **Amazon API Gateway**: 
  - 1M API calls per month for 12 months
  - After 12 months, implement direct Lambda function URLs as fallback

- **Amazon S3**: 
  - 5GB standard storage
  - 20,000 GET requests per month
  - 2,000 PUT requests per month
  - 12 months free, then minimal cost for 5GB

- **Amazon DynamoDB**: 
  - 25GB storage
  - 25 read capacity units (RCU) and 25 write capacity units (WCU)
  - Always free

- **AWS Transcribe**: 
  - 60 minutes free per month for first 12 months
  - Critical bottleneck - requires careful usage management

- **AWS Polly**: 
  - 5M characters per month for speech synthesis
  - Always free for standard voices

- **AWS Translate**: 
  - 2M characters per month for 12 months
  - Used for translating between Indian languages

- **Amazon Comprehend**: 
  - 50,000 units per month for 12 months
  - Used for text analysis and entity extraction (alternative to Bedrock)

- **CloudWatch**: 
  - 10 custom metrics and 10 alarms
  - 5GB log ingestion and storage
  - Always free

### AWS Services Justification

- **AWS Lambda**: Serverless compute with generous free tier, perfect for intermittent workloads
- **API Gateway**: Managed API service with 1M free calls/month (sufficient for ~33 requests/day)
- **Amazon S3**: 5GB free storage sufficient for hundreds of scheme documents
- **DynamoDB**: 25GB free storage with 25 RCU/WCU handles moderate traffic
- **AWS Transcribe**: 60 minutes/month = ~2 minutes/day (requires caching and optimization)
- **AWS Polly**: 5M characters/month = ~166K characters/day (generous for voice responses)
- **AWS Translate**: 2M characters/month for multi-language support
- **Amazon Comprehend**: Alternative to Bedrock for text analysis within free tier
- **CloudWatch**: Free tier sufficient for basic monitoring and alerting

## Components and Interfaces

### 1. API Gateway

**Responsibilities**:
- Accept incoming HTTP/HTTPS requests with audio payload
- Authenticate requests using API keys or AWS Cognito
- Route requests to the Request Orchestrator Lambda
- Apply rate limiting and throttling
- Return responses to clients

**Interfaces**:
- **Input**: HTTP POST request with audio file (MP3/WAV), optional user ID, language preference
- **Output**: HTTP response with audio file (MP3), status code, error messages

**Configuration**:
- REST API with regional endpoint for lower latency
- Request timeout: 29 seconds (API Gateway maximum)
- Throttling: 1000 requests per second per user
- CORS enabled for web client access

### 2. Request Orchestrator Lambda

**Responsibilities**:
- Coordinate the entire request-response workflow
- Invoke specialized Lambda functions in sequence
- Handle errors and implement retry logic
- Manage state across processing steps
- Return final response to API Gateway

**Interfaces**:
- **Input**: Event from API Gateway containing audio data, user ID, metadata
- **Output**: Processed response with audio, text summary, status
- **Invokes**: Speech-to-Text Lambda, Scheme Retriever Lambda, AI Summarizer Lambda, Text-to-Speech Lambda, History Manager Lambda

**Configuration**:
- Runtime: Python 3.11 or Node.js 18.x
- Memory: 256 MB (reduced from 512 MB to save compute time)
- Timeout: 25 seconds
- Provisioned concurrency: 0 (disabled to stay in free tier)
- Environment variables: S3 bucket name, DynamoDB table name, cache configuration
- **Free Tier Impact**: Each invocation consumes GB-seconds (memory × duration)

### 3. Speech-to-Text Lambda

**Responsibilities**:
- Convert audio input to text
- Detect language (Hindi or Marathi)
- Handle audio format conversion if needed
- Return transcribed text with language metadata

**Interfaces**:
- **Input**: Audio file (base64 encoded or S3 reference), optional language hint
- **Output**: Transcribed text, detected language, confidence score
- **External Service**: AWS Transcribe

**Implementation Details**:
- Use AWS Transcribe StartTranscriptionJob for async processing
- For low latency, use StartStreamTranscription for real-time processing
- **Phase 1 (MVP)**: Support Hindi (hi-IN) and Marathi (mr-IN) as specified in requirements
- **Phase 2 (Future)**: Expand to additional Indian languages:
  - Tamil (ta-IN), Telugu (te-IN), Bengali (bn-IN)
  - Kannada (kn-IN), Malayalam (ml-IN), Gujarati (gu-IN)
  - Punjabi (pa-IN), Odia (or-IN), Assamese (as-IN)
- Implement audio preprocessing for noise reduction if needed
- **Free Tier Optimization**: Cache transcriptions aggressively (60 min/month = ~2 min/day limit)

**Configuration**:
- Runtime: Python 3.11
- Memory: 128 MB (reduced to minimize compute cost)
- Timeout: 10 seconds
- IAM permissions: transcribe:StartTranscriptionJob, transcribe:GetTranscriptionJob
- **Free Tier Impact**: 60 minutes/month limit - CRITICAL BOTTLENECK

### 4. Scheme Retriever Lambda

**Responsibilities**:
- Parse user query to identify relevant keywords
- Search S3 for matching scheme documents
- Retrieve document content
- Return relevant scheme information

**Interfaces**:
- **Input**: Transcribed query text, language
- **Output**: List of relevant scheme documents with content
- **External Service**: Amazon S3

**Implementation Details**:
- Use S3 object tagging and metadata for efficient search
- Implement keyword extraction from query text
- Support fuzzy matching for scheme names
- Cache frequently accessed documents in DynamoDB (not Lambda memory due to cold starts)
- Use S3 Select for filtering large documents
- **Free Tier Optimization**: Minimize S3 GET requests (20K/month limit)

**Configuration**:
- Runtime: Python 3.11
- Memory: 256 MB (reduced from 512 MB)
- Timeout: 8 seconds
- IAM permissions: s3:GetObject, s3:ListBucket
- **Free Tier Impact**: Each retrieval consumes S3 GET requests

**S3 Bucket Structure**:
```
schemes/
  ├── hindi/
  │   ├── scheme-1.json
  │   └── scheme-2.json
  ├── marathi/
  │   ├── scheme-1.json
  │   └── scheme-2.json
  ├── [future-languages]/
  │   └── [future expansion for Tamil, Telugu, Bengali, etc.]
  └── metadata.json
```

### 5. AI Summarizer Lambda

**Responsibilities**:
- Generate simplified summaries of scheme documents
- Maintain language consistency (Hindi and Marathi in Phase 1)
- Focus on key benefits, eligibility, and application process
- Handle summarization errors gracefully

**Interfaces**:
- **Input**: Scheme documents, user query, language
- **Output**: Simplified summary text in the specified language
- **External Service**: Amazon Comprehend (for entity extraction), AWS Translate (for language support)

**Implementation Details**:
- **Note**: Requirements specify Amazon Bedrock (Requirement 4.1), but Bedrock is not available in AWS free tier
- **Free Tier Alternative**: Use Amazon Comprehend (50K units/month free) for entity extraction combined with rule-based extractive summarization
- Use Amazon Comprehend for entity extraction and key phrase detection
- Implement rule-based extractive summarization algorithm
- Use AWS Translate to convert summaries between Hindi and Marathi if needed
- Construct summaries by extracting key sentences based on:
  - Scheme name and description
  - Eligibility criteria
  - Benefits list
  - Application process
- Limit summary length to 200-300 words for TTS efficiency
- Implement template-based summarization for consistency
- **Future Enhancement**: Upgrade to Amazon Bedrock when budget allows for higher-quality abstractive summarization

**Summarization Algorithm**:
```
1. Extract entities using Comprehend (scheme name, amounts, dates)
2. Identify key phrases related to eligibility, benefits, application
3. Extract sentences containing these key phrases
4. Rank sentences by relevance score
5. Select top 5-7 sentences
6. Arrange in logical order (what → who → benefits → how)
7. Translate to target language if needed using AWS Translate
```

**Configuration**:
- Runtime: Python 3.11
- Memory: 512 MB (needed for text processing)
- Timeout: 15 seconds
- IAM permissions: comprehend:DetectEntities, comprehend:DetectKeyPhrases, translate:TranslateText
- **Free Tier Impact**: 
  - Comprehend: 50K units/month (1 unit = 100 characters)
  - Translate: 2M characters/month

### 6. Text-to-Speech Lambda

**Responsibilities**:
- Convert summary text to speech audio
- Use appropriate voice for the language
- Optimize audio format for low bandwidth
- Return audio file

**Interfaces**:
- **Input**: Summary text, language
- **Output**: Audio file (MP3), audio metadata
- **External Service**: AWS Polly

**Implementation Details**:
- Use AWS Polly SynthesizeSpeech API
- **Phase 1 (MVP)**: Support Hindi and Marathi voices
  - Hindi: Aditi (female, neural), Kajal (female, standard)
  - Marathi: Use Hindi voice (Aditi) as closest match
- **Phase 2 (Future)**: Expand to additional language voices
  - Tamil, Telugu, Bengali, Kannada, Malayalam, Gujarati voices
  - Punjabi: Use Hindi voice as fallback
  - Odia: Use Bengali voice as fallback
  - Assamese: Use Bengali voice as fallback
- Output format: MP3 at 24 kbps for bandwidth optimization
- Use standard voices (not neural) to stay within free tier
- Implement SSML for better pronunciation if needed
- **Free Tier Optimization**: 5M characters/month = ~166K characters/day (generous limit)

**Configuration**:
- Runtime: Python 3.11
- Memory: 128 MB (reduced from 256 MB)
- Timeout: 8 seconds
- IAM permissions: polly:SynthesizeSpeech
- **Free Tier Impact**: 5M characters/month for standard voices (always free)

### 7. History Manager Lambda

**Responsibilities**:
- Store user interaction history
- Associate interactions with user ID and timestamp
- Store query, response, and language metadata
- Support future analytics and personalization

**Interfaces**:
- **Input**: User ID, query text, response text, language, timestamp
- **Output**: Success/failure status
- **External Service**: DynamoDB

**Implementation Details**:
- Use DynamoDB PutItem for storing history
- Partition key: user_id
- Sort key: timestamp (ISO 8601 format)
- Attributes: query_text, response_text, language, session_id
- Implement TTL for automatic data expiration (e.g., 90 days)
- Use batch writes for efficiency if storing multiple records

**Configuration**:
- Runtime: Python 3.11
- Memory: 128 MB
- Timeout: 5 seconds
- IAM permissions: dynamodb:PutItem
- **Free Tier Impact**: 25 WCU allows ~2.5M writes/month (generous for history storage)

**DynamoDB Table Schema**:
```
Table: sevasetu-user-history
Partition Key: user_id (String)
Sort Key: timestamp (String)
Attributes:
  - query_text (String)
  - response_text (String)
  - language (String)
  - session_id (String)
  - ttl (Number) - Unix timestamp for expiration
Billing Mode: On-Demand (within free tier limits)
```

### 8. Cache Manager Lambda

**Responsibilities**:
- Cache transcriptions to reduce AWS Transcribe usage
- Cache scheme documents to reduce S3 GET requests
- Cache summaries for identical queries
- Manage cache expiration and invalidation

**Interfaces**:
- **Input**: Cache key, cache value, TTL
- **Output**: Cached value or cache miss indicator
- **External Service**: DynamoDB (used as cache store)

**Implementation Details**:
- Use DynamoDB as distributed cache (within 25GB free tier)
- Cache transcriptions with 24-hour TTL
- Cache scheme documents with 7-day TTL
- Cache summaries with 1-hour TTL
- Implement cache key hashing for efficient lookups
- Use DynamoDB TTL for automatic expiration

**Caching Strategy**:
- **Transcription Cache**: Hash of audio file → transcribed text
- **Scheme Cache**: Scheme ID → scheme document
- **Summary Cache**: Hash of (query + scheme IDs + language) → summary
- **Query Cache**: Hash of query text → list of relevant scheme IDs

**Configuration**:
- Runtime: Python 3.11
- Memory: 128 MB
- Timeout: 3 seconds
- IAM permissions: dynamodb:GetItem, dynamodb:PutItem, dynamodb:Query
- **Free Tier Impact**: Uses DynamoDB 25GB storage and 25 RCU/WCU

**DynamoDB Cache Table Schema**:
```
Table: sevasetu-cache
Partition Key: cache_type (String) - "transcription", "scheme", "summary", "query"
Sort Key: cache_key (String) - hashed key
Attributes:
  - cache_value (String or Binary)
  - created_at (Number)
  - ttl (Number) - Unix timestamp for expiration
  - hit_count (Number) - for cache analytics
Billing Mode: On-Demand (within free tier limits)
```

## Data Models

### Request Model

```json
{
  "audio": "base64_encoded_audio_data",
  "user_id": "string",
  "language_preference": "hi-IN | mr-IN | auto",
  "session_id": "string",
  "metadata": {
    "device_type": "string",
    "app_version": "string"
  }
}
```

### Response Model

```json
{
  "status": "success | error",
  "audio": "base64_encoded_audio_data",
  "text_summary": "string",
  "language": "hi-IN | mr-IN",
  "schemes_found": ["string"],
  "cached": "boolean",
  "error": {
    "code": "string",
    "message": "string"
  }
}
```

### Scheme Document Model

```json
{
  "scheme_id": "string",
  "scheme_name": {
    "hindi": "string",
    "marathi": "string",
    "english": "string"
  },
  "description": {
    "hindi": "string",
    "marathi": "string"
  },
  "eligibility": {
    "hindi": ["string"],
    "marathi": ["string"]
  },
  "benefits": {
    "hindi": ["string"],
    "marathi": ["string"]
  },
  "application_process": {
    "hindi": "string",
    "marathi": "string"
  },
  "keywords": ["string"],
  "category": "string",
  "last_updated": "ISO8601 timestamp"
}
```

### User History Model

```json
{
  "user_id": "string",
  "timestamp": "ISO8601 timestamp",
  "query_text": "string",
  "response_text": "string",
  "language": "hi-IN | mr-IN",
  "session_id": "string",
  "schemes_retrieved": ["string"],
  "cached": "boolean",
  "ttl": 1234567890
}
```

## Data Flow

### End-to-End Request Flow

1. **User Initiates Query**:
   - User speaks question in any supported Indian language
   - Mobile/web app captures audio
   - App sends HTTP POST to API Gateway with audio payload

2. **API Gateway Processing**:
   - Validates authentication token
   - Checks rate limits
   - Routes request to Request Orchestrator Lambda

3. **Cache Check**:
   - Orchestrator computes hash of audio file
   - Checks Cache Manager for existing transcription
   - If cache hit, skip transcription step (saves Transcribe minutes)

4. **Speech-to-Text Conversion** (if not cached):
   - Orchestrator invokes Speech-to-Text Lambda
   - Lambda calls AWS Transcribe with audio
   - Transcribe returns text and detected language
   - Result is cached in DynamoDB for 24 hours
   - Language is stored for subsequent steps

5. **Query Cache Check**:
   - Orchestrator checks Cache Manager for query hash
   - If cache hit, retrieve cached scheme IDs and skip retrieval

6. **Scheme Retrieval** (if not cached):
   - Orchestrator invokes Scheme Retriever Lambda with query text
   - Lambda extracts keywords from query
   - Lambda checks Cache Manager for scheme documents
   - If scheme not cached, retrieves from S3 bucket
   - Caches scheme documents in DynamoDB for 7 days
   - Returns relevant scheme documents

7. **Summary Cache Check**:
   - Orchestrator computes hash of (query + scheme IDs + language)
   - Checks Cache Manager for existing summary
   - If cache hit, skip summarization (saves Comprehend/Translate usage)

8. **AI Summarization** (if not cached):
   - Orchestrator invokes AI Summarizer Lambda
   - Lambda uses Amazon Comprehend for entity extraction
   - Lambda constructs extractive summary
   - If needed, uses AWS Translate for language conversion
   - Caches summary in DynamoDB for 1 hour
   - Returns simplified summary in user's language

9. **Text-to-Speech Conversion**:
   - Orchestrator invokes Text-to-Speech Lambda
   - Lambda calls AWS Polly with summary text
   - Polly returns audio file in MP3 format (24 kbps)

10. **History Storage**:
    - Orchestrator invokes History Manager Lambda (async)
    - Lambda stores interaction in DynamoDB
    - Storage happens in background, doesn't block response

11. **Response Delivery**:
    - Orchestrator returns audio, text, and cache status to API Gateway
    - API Gateway sends HTTP response to client
    - Client plays audio response to user

### Cache Hit Flow (Optimized Path)

For frequently asked questions:
1. User query → API Gateway → Orchestrator
2. Orchestrator checks transcription cache → Cache hit
3. Orchestrator checks query cache → Cache hit (scheme IDs)
4. Orchestrator checks summary cache → Cache hit
5. Orchestrator invokes TTS Lambda (Polly)
6. Response delivered to user

**Free Tier Savings**: Cache hits eliminate Transcribe, S3, Comprehend, and Translate usage

### Error Flow

- If any step fails, Orchestrator implements retry logic (up to 3 attempts)
- If retries fail, Orchestrator generates error message in user's language
- Error message is converted to speech and returned to user
- All errors are logged to CloudWatch for monitoring

## Security Considerations

### Free Tier Cost Optimization and Monitoring

**Critical Free Tier Limits**:

1. **AWS Transcribe (MOST CRITICAL)**:
   - Limit: 60 minutes/month (first 12 months)
   - Daily budget: ~2 minutes/day
   - Strategy: Aggressive caching of transcriptions (24-hour TTL)
   - Monitoring: CloudWatch alarm at 50 minutes/month
   - Fallback: Disable new transcriptions when limit reached, serve cached responses only

2. **API Gateway**:
   - Limit: 1M calls/month (first 12 months)
   - Daily budget: ~33,000 calls/day
   - Strategy: Rate limiting per user (10 requests/hour)
   - Monitoring: CloudWatch alarm at 900K calls/month
   - Fallback: Switch to Lambda Function URLs after 12 months

3. **S3 GET Requests**:
   - Limit: 20,000 GET requests/month
   - Daily budget: ~666 requests/day
   - Strategy: Cache scheme documents in DynamoDB (7-day TTL)
   - Monitoring: CloudWatch alarm at 18,000 requests/month
   - Optimization: Batch multiple scheme retrievals into single S3 call

4. **AWS Translate**:
   - Limit: 2M characters/month (first 12 months)
   - Daily budget: ~66,000 characters/day
   - Strategy: Pre-translate scheme documents, cache translations
   - Monitoring: CloudWatch alarm at 1.8M characters/month
   - Fallback: Serve content in original language only

5. **Amazon Comprehend**:
   - Limit: 50,000 units/month (first 12 months)
   - 1 unit = 100 characters
   - Daily budget: ~1,666 units/day (~166,600 characters)
   - Strategy: Cache entity extraction results
   - Monitoring: CloudWatch alarm at 45,000 units/month

6. **Lambda**:
   - Limit: 400,000 GB-seconds/month (always free)
   - Strategy: Minimize memory allocation, optimize execution time
   - Monitoring: CloudWatch alarm at 350,000 GB-seconds/month
   - Optimization: Use 128-256 MB memory where possible

7. **DynamoDB**:
   - Limit: 25 RCU/WCU, 25GB storage (always free)
   - Strategy: Use on-demand billing, implement TTL for cache cleanup
   - Monitoring: CloudWatch alarm at 20GB storage
   - Optimization: Compress cached data, aggressive TTL

8. **AWS Polly**:
   - Limit: 5M characters/month (always free for standard voices)
   - Daily budget: ~166,000 characters/day
   - Strategy: Use standard voices only (not neural)
   - Monitoring: CloudWatch alarm at 4.5M characters/month
   - Optimization: Limit summary length to 300 words

### Caching Strategy for Free Tier Optimization

**Three-Tier Caching**:

1. **Transcription Cache** (DynamoDB):
   - Key: SHA-256 hash of audio file
   - Value: Transcribed text + language
   - TTL: 24 hours
   - Impact: Saves Transcribe minutes (most critical)

2. **Scheme Document Cache** (DynamoDB):
   - Key: Scheme ID
   - Value: Full scheme document
   - TTL: 7 days
   - Impact: Saves S3 GET requests

3. **Summary Cache** (DynamoDB):
   - Key: Hash of (query + scheme IDs + language)
   - Value: Generated summary
   - TTL: 1 hour
   - Impact: Saves Comprehend and Translate usage

**Cache Warming**:
- Pre-populate cache with top 20 most common queries
- Pre-cache all scheme documents during off-peak hours
- Reduces cold-path executions

### Request Throttling

**User-Level Throttling**:
- Maximum 10 requests per user per hour
- Maximum 3 requests per user per minute
- Implemented in API Gateway using usage plans

**System-Level Throttling**:
- When Transcribe usage > 50 minutes/month: Reduce to 5 requests/user/hour
- When Transcribe usage > 55 minutes/month: Serve cached responses only
- When API Gateway > 900K calls/month: Reduce to 3 requests/user/hour

### Usage Monitoring and Alerts

**CloudWatch Alarms**:

```python
# Transcribe Usage (CRITICAL)
Alarm: TranscribeUsageHigh
Metric: Custom metric tracking minutes used
Threshold: 50 minutes/month (83% of limit)
Action: SNS notification + reduce throttling limits

Alarm: TranscribeUsageCritical
Metric: Custom metric tracking minutes used
Threshold: 55 minutes/month (92% of limit)
Action: SNS notification + disable new transcriptions

# API Gateway Usage
Alarm: APIGatewayUsageHigh
Metric: Count of API calls
Threshold: 900,000 calls/month (90% of limit)
Action: SNS notification + increase rate limiting

# S3 GET Requests
Alarm: S3GetRequestsHigh
Metric: NumberOfRequests (GET)
Threshold: 18,000 requests/month (90% of limit)
Action: SNS notification + increase cache TTL

# Lambda Compute
Alarm: LambdaComputeHigh
Metric: Custom metric tracking GB-seconds
Threshold: 350,000 GB-seconds/month (87.5% of limit)
Action: SNS notification + optimize memory allocation

# DynamoDB Storage
Alarm: DynamoDBStorageHigh
Metric: AccountProvisionedReadCapacityUtilization
Threshold: 20GB (80% of limit)
Action: SNS notification + reduce cache TTL
```

### Cost Dashboard

**Custom CloudWatch Dashboard**:
- Real-time usage metrics for all free tier services
- Percentage of monthly limit consumed
- Projected end-of-month usage
- Cache hit rates
- Request throttling statistics

### Graceful Degradation Strategy

**When limits are approached**:

1. **Phase 1 (80% of limit)**:
   - Increase cache TTL
   - Reduce user request limits
   - Send warning notifications

2. **Phase 2 (90% of limit)**:
   - Aggressive caching (extend TTL to 7 days)
   - Serve cached responses preferentially
   - Disable non-critical features

3. **Phase 3 (95% of limit)**:
   - Serve cached responses only
   - Disable new transcriptions
   - Display maintenance message to users

### Data Compression

**Reduce Storage and Transfer Costs**:
- Compress cached data in DynamoDB using gzip
- Use compact JSON format (no whitespace)
- Store audio hashes instead of full audio files
- Compress CloudWatch logs

### Efficient Data Access Patterns

**DynamoDB Optimization**:
- Use batch operations (BatchGetItem, BatchWriteItem)
- Implement efficient query patterns with proper keys
- Use projection expressions to retrieve only needed attributes
- Enable DynamoDB TTL for automatic cleanup

**S3 Optimization**:
- Use S3 Select to filter data server-side
- Batch multiple scheme documents into single files
- Use S3 object tagging for efficient search without GET requests

### Lambda Optimization

**Reduce Compute Time**:
- Minimize cold starts (keep functions small)
- Reuse connections and clients across invocations
- Use environment variables for configuration
- Optimize code for performance (avoid unnecessary processing)
- Use ARM-based Graviton2 processors (20% faster, same cost)

**Memory Optimization**:
- Start with 128 MB and increase only if needed
- Monitor actual memory usage with CloudWatch
- Right-size memory allocation per function

### Post-Free-Tier Strategy

**After 12 months (when some services expire)**:

1. **API Gateway → Lambda Function URLs**:
   - Switch from API Gateway to Lambda Function URLs (always free)
   - Implement authentication in Lambda code

2. **Transcribe → Client-Side Speech Recognition**:
   - Explore browser-based Web Speech API
   - Use open-source speech recognition libraries
   - Pre-process audio on client side

3. **Translate → Pre-Translated Content**:
   - Pre-translate all scheme documents
   - Store translations in S3
   - Eliminate runtime translation

4. **Comprehend → Rule-Based Extraction**:
   - Implement custom entity extraction
   - Use regex and keyword matching
   - Eliminate Comprehend dependency

## Security Considerations

### Authentication and Authorization

- **API Gateway**: Use AWS Cognito User Pools for user authentication
- **Service-to-Service**: Use IAM roles for Lambda-to-service communication
- **Principle of Least Privilege**: Each Lambda has minimal IAM permissions

### Data Encryption

- **In Transit**: All data encrypted using TLS 1.2 or higher
- **At Rest**: 
  - S3 uses server-side encryption (SSE-S3 or SSE-KMS)
  - DynamoDB uses encryption at rest with AWS managed keys
  - Lambda environment variables encrypted with KMS

### Data Privacy

- **Audio Data**: Not stored permanently, only processed in memory
- **User History**: Stored with TTL for automatic expiration
- **PII Handling**: Minimal PII collection, user_id is anonymized identifier
- **Compliance**: Design supports GDPR-like data deletion requests

### Network Security

- **VPC**: Lambda functions can be deployed in VPC for additional isolation
- **Security Groups**: Restrict outbound traffic to required AWS services
- **API Gateway**: Enable AWS WAF for protection against common attacks

### Monitoring and Auditing

- **CloudWatch Logs**: All Lambda invocations logged
- **CloudTrail**: API calls to AWS services audited
- **Alarms**: Set up for unusual activity patterns

## Scalability Considerations

### Free Tier Constraints

The system is designed to operate within AWS free tier limits, which imposes specific scalability constraints:

**Hard Limits**:
- Maximum ~2 minutes of transcription per day (60 min/month)
- Maximum ~33,000 API calls per day (1M/month for 12 months)
- Maximum ~666 S3 GET requests per day (20K/month)
- These limits define the maximum scale of the system

**Estimated User Capacity**:
- Average query duration: 30 seconds
- Transcribe limit: 60 minutes/month = 120 queries/month without caching
- With 80% cache hit rate: 600 queries/month
- Sustainable user base: ~20 active users (30 queries/month each)

### Horizontal Scaling (Within Free Tier)

- **Lambda**: Automatically scales to handle concurrent requests (up to 1000 concurrent executions)
- **API Gateway**: Handles high request volume (throttled to stay within free tier)
- **DynamoDB**: On-demand capacity mode scales automatically (within 25 RCU/WCU free tier)
- **S3**: Unlimited scalability for reads (throttled to 20K GET/month)

### Performance Optimization

- **Lambda Provisioned Concurrency**: DISABLED (not in free tier)
- **Connection Pooling**: Reuse connections to DynamoDB and other services
- **Caching**: 
  - Aggressive caching to maximize free tier efficiency
  - DynamoDB as distributed cache (within 25GB free tier)
  - Target 80%+ cache hit rate to stay within limits

### Cost Optimization

- **Lambda**: Use ARM-based Graviton2 processors (20% better performance, same free tier)
- **S3**: Use S3 Intelligent-Tiering (not applicable for 5GB free tier)
- **DynamoDB**: On-demand pricing within free tier limits
- **Comprehend/Translate**: Aggressive caching to minimize usage

### Resource Limits

- **Lambda Timeout**: Set appropriate timeouts to prevent runaway executions
- **API Gateway Timeout**: 29 seconds maximum
- **DynamoDB**: Implement exponential backoff for throttling
- **User Throttling**: 10 requests/hour per user to stay within limits

### Scaling Beyond Free Tier

If the system needs to scale beyond free tier limits:

1. **Paid Tier Migration**:
   - Enable paid usage for Transcribe, API Gateway, Translate
   - Estimated cost: $0.024/minute (Transcribe) + $3.50/million requests (API Gateway)
   - For 1000 users: ~$50-100/month

2. **Alternative Architecture**:
   - Replace Transcribe with open-source alternatives (Whisper, Vosk)
   - Replace API Gateway with Lambda Function URLs (always free)
   - Replace Translate with pre-translated content
   - Replace Comprehend with rule-based extraction

3. **Hybrid Approach**:
   - Keep free tier for cached responses
   - Use paid tier only for cache misses
   - Optimize cache hit rate to minimize paid usage

## Latency Optimization Strategies

### Target Latency Budget

- Total end-to-end latency target: < 10 seconds (increased due to free tier constraints)
- Breakdown:
  - Speech-to-Text: 2-4 seconds (or 0s if cached)
  - Scheme Retrieval: 0.5-1 second (or 0.1s if cached)
  - AI Summarization: 3-5 seconds (or 0s if cached)
  - Text-to-Speech: 1-2 seconds
  - Network overhead: 0.5-1 second
  - Cache lookup: 0.1-0.2 seconds per check

### Optimization Techniques

1. **Maximize Cache Hits**:
   - Target 80%+ cache hit rate for transcriptions
   - Pre-warm cache with common queries
   - Extend cache TTL during low-usage periods
   - Cache hit reduces latency from 10s to 3s

2. **Reduce Cold Starts**:
   - Keep Lambda packages small (< 10 MB)
   - Use Lambda layers for shared dependencies
   - Minimize dependencies in deployment package
   - Note: Provisioned concurrency disabled (not in free tier)

3. **Parallel Processing**:
   - Invoke History Manager Lambda asynchronously
   - Check multiple cache layers in parallel
   - Batch DynamoDB operations where possible

4. **Audio Optimization**:
   - Use compressed audio formats (MP3 at 24 kbps)
   - Support progressive audio playback
   - Minimize audio file size for faster transfer

5. **Network Optimization**:
   - Deploy Lambda in same region as other AWS services
   - Use regional API Gateway endpoints
   - Implement HTTP/2 for multiplexing

6. **Caching Strategy**:
   - Three-tier cache (transcription, scheme, summary)
   - DynamoDB as distributed cache (within free tier)
   - Cache hit rate monitoring and optimization

7. **Database Optimization**:
   - Use efficient DynamoDB query patterns
   - Use batch operations where possible
   - Implement projection expressions to retrieve only needed data
   - Note: DynamoDB DAX disabled (not in free tier)

### Cache Performance Impact

**Without Cache** (Cold Path):
- Total latency: ~10 seconds
- Transcribe: 3s + Retrieval: 1s + Summarization: 4s + TTS: 2s

**With Cache** (Warm Path):
- Total latency: ~3 seconds
- Cache lookup: 0.2s + TTS: 2s + Network: 0.8s
- 70% latency reduction

### Free Tier Trade-offs

- **No Provisioned Concurrency**: Cold starts add 1-3 seconds on first request
- **No DynamoDB DAX**: Cache lookups take 10-50ms instead of <1ms
- **No CloudFront CDN**: No edge caching for static assets
- **Aggressive Caching**: Slightly stale data (acceptable for government schemes)

## Error Handling

### Error Categories

1. **User Errors**:
   - Unclear audio input
   - Unsupported language
   - Invalid request format

2. **System Errors**:
   - Service unavailability (Transcribe, Bedrock, Polly)
   - Timeout errors
   - Rate limiting

3. **Data Errors**:
   - Scheme not found
   - Corrupted scheme documents
   - Database write failures

### Error Handling Strategy

- **Retry Logic**: Exponential backoff with jitter for transient failures
- **Circuit Breaker**: Fail fast when service is consistently unavailable
- **Graceful Degradation**: Return partial results when possible
- **User-Friendly Messages**: Convert technical errors to simple language
- **Logging**: Log all errors with context for debugging

### Error Response Format

```json
{
  "status": "error",
  "error": {
    "code": "TRANSCRIPTION_FAILED",
    "message": "कृपया फिर से बोलें (Please speak again)",
    "user_message": "हम आपकी आवाज़ नहीं सुन पाए। कृपया फिर से कोशिश करें।",
    "retry_allowed": true
  }
}
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: Audio Input Acceptance

*For any* audio input in a supported language (Hindi or Marathi), the Voice Input Handler should accept the audio data without rejection.

**Validates: Requirements 1.1**

### Property 2: Audio Format Compatibility

*For any* audio format commonly used by mobile devices (MP3, WAV, AAC), the Voice Input Handler should successfully process the audio.

**Validates: Requirements 1.2**

### Property 3: Speech-to-Text Language Preservation

*For any* audio input in a supported language, when converted to text and the language detected, the output language should match the input language.

**Validates: Requirements 1.3, 1.5**

### Property 4: Low-Quality Audio Error Handling

*For any* audio input with insufficient quality (high noise, low volume, corrupted data), the Speech-to-Text Service should return an error message requesting the user to repeat.

**Validates: Requirements 1.4**

### Property 5: Language Detection Accuracy

*For any* transcribed query in Hindi or Marathi, the system should correctly detect which supported language was used.

**Validates: Requirements 2.1**

### Property 6: End-to-End Language Consistency

*For any* user query in a supported language, the language should remain consistent throughout the entire request-response cycle (transcription → retrieval → summarization → TTS).

**Validates: Requirements 2.2, 2.3**

### Property 7: Scheme Retrieval Completeness

*For any* user query with known matching scheme documents, the Scheme Retriever should return all relevant documents without omission.

**Validates: Requirements 3.1, 3.2, 3.3**

### Property 8: AI Summarization Invocation

*For any* set of retrieved scheme documents, the AI Summarizer should process them to generate a summary using Amazon Comprehend for entity extraction combined with rule-based extractive summarization.

**Validates: Requirements 4.1**

**Note**: Requirement 4.1 specifies Amazon Bedrock, but this design uses Amazon Comprehend (free tier) as a cost-effective alternative. The property validates that AI-powered summarization occurs, regardless of the specific service used.

### Property 9: Summary Language Consistency

*For any* scheme documents and user query language, the generated summary should be in the same language as the query.

**Validates: Requirements 4.3**

### Property 10: Summary Content Completeness

*For any* scheme document, the generated summary should contain information about benefits, eligibility criteria, and application process.

**Validates: Requirements 4.4**

### Property 11: Text-to-Speech Conversion and Delivery

*For any* generated summary text in a supported language, the system should convert it to speech in the same language and deliver the audio response.

**Validates: Requirements 5.1, 5.4**

### Property 12: Audio Format Optimization

*For any* generated audio response, the format should be optimized for low bandwidth (MP3 format at ≤ 24 kbps bitrate).

**Validates: Requirements 5.2**

### Property 13: User History Persistence

*For any* processed user query and generated response, the History Manager should store a record in the database containing the query text, response text, user identifier, timestamp, and language.

**Validates: Requirements 6.1, 6.2, 6.3, 6.4**

### Property 14: Audio Compression

*For any* audio data transmitted, the data should be compressed to reduce bandwidth usage.

**Validates: Requirements 8.2**

### Property 15: Scheme Document Caching

*For any* scheme document accessed multiple times, subsequent accesses should retrieve from cache rather than S3 (cache hit).

**Validates: Requirements 8.4**

### Property 16: Authentication Enforcement

*For any* incoming request to the API Gateway, the request should be rejected if it lacks valid authentication credentials.

**Validates: Requirements 9.1**

### Property 17: Audio Data Non-Persistence

*For any* audio file processed during a request, the audio should not exist in storage after the request completes.

**Validates: Requirements 9.6**

### Property 18: Retry Logic on Failure

*For any* component failure during processing, the system should retry the operation up to 3 times before giving up.

**Validates: Requirements 10.4**

### Property 19: Error Message Language Consistency

*For any* error that occurs during processing, the error message should be generated in the same language as the user's query.

**Validates: Requirements 12.1**

### Property 20: Error Message Guidance

*For any* error message, the message should contain actionable guidance text (not just error codes).

**Validates: Requirements 12.2**

### Property 21: Error Logging

*For any* error that occurs, the error should be logged to CloudWatch with sufficient context for debugging.

**Validates: Requirements 12.3**

### Property 22: Error Type Distinction

*For any* error, the error message should clearly indicate whether it is a user error (e.g., unclear audio) or a system error (e.g., service unavailable).

**Validates: Requirements 12.5**

## Testing Strategy

### Dual Testing Approach

The SevaSetu system requires both unit testing and property-based testing for comprehensive coverage:

- **Unit Tests**: Verify specific examples, edge cases, error conditions, and integration points
- **Property Tests**: Verify universal properties across all inputs through randomized testing

Both approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide range of inputs.

### Property-Based Testing Configuration

**Library Selection**:
- **Python**: Use `hypothesis` library for property-based testing
- **Node.js/TypeScript**: Use `fast-check` library for property-based testing

**Test Configuration**:
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `# Feature: sevasetu, Property {number}: {property_text}`

**Property Test Implementation**:
- Each correctness property listed above must be implemented as a single property-based test
- Tests should generate random valid inputs within the domain
- Tests should verify the property holds for all generated inputs

### Unit Testing Strategy

**Focus Areas**:
1. **Specific Examples**: Test known good inputs and expected outputs
2. **Edge Cases**: Empty queries, maximum length inputs, special characters, all 11 languages
3. **Error Conditions**: Service failures, timeouts, invalid data, free tier limit exceeded
4. **Integration Points**: Lambda-to-Lambda communication, AWS service interactions
5. **Cache Logic**: Cache hits, cache misses, cache expiration, cache invalidation

**Test Coverage Goals**:
- 80% code coverage minimum
- 100% coverage for error handling paths
- 100% coverage for cache logic
- All AWS service integrations mocked for unit tests

### Integration Testing

**End-to-End Tests**:
- Test complete flow from audio input to audio output
- Use real AWS services in test environment (within free tier limits)
- **Phase 1**: Verify language consistency across pipeline for Hindi and Marathi
- **Phase 2**: Expand testing to additional languages (Tamil, Telugu, Bengali, Kannada, Malayalam, Gujarati, Punjabi, Odia, Assamese)
- Test with actual audio samples in Hindi and Marathi
- Test cache hit and cache miss scenarios

**Performance Tests**:
- Measure latency for each component
- Verify total response time < 10 seconds (cold path)
- Verify total response time < 3 seconds (warm path with cache)
- Test with simulated 2G network conditions

**Free Tier Compliance Tests**:
- Monitor usage of all AWS services during testing
- Verify caching reduces Transcribe usage by 80%+
- Verify S3 GET requests are minimized through caching
- Test throttling behavior when limits are approached

### Test Data

**Audio Samples**:
- Collect diverse voice samples in Hindi and Marathi (Phase 1)
- Include various accents and speaking speeds
- Include low-quality audio for error testing
- Test with different audio formats (MP3, WAV, AAC)
- **Phase 2**: Expand to additional language audio samples

**Scheme Documents**:
- Create test scheme documents in Hindi and Marathi
- Include edge cases (very long, very short, missing fields)
- Test with real government scheme data

**Mock Services**:
- Mock AWS Transcribe responses for Hindi and Marathi
- Mock Amazon Comprehend responses
- Mock AWS Translate responses for Hindi-Marathi language pairs
- Mock AWS Polly responses for Hindi and Marathi
- Use LocalStack for local AWS service testing

### Cache Testing

**Cache Hit Rate Testing**:
- Measure cache hit rate for transcriptions
- Measure cache hit rate for scheme documents
- Measure cache hit rate for summaries
- Target: 80%+ cache hit rate in production

**Cache Invalidation Testing**:
- Test TTL expiration
- Test manual cache invalidation
- Test cache size limits

### Continuous Testing

- Run unit tests on every commit
- Run property tests on every pull request
- Run integration tests nightly (monitor free tier usage)
- Run performance tests weekly
- Run free tier compliance tests daily

## Future Enhancements

### Language Expansion Roadmap

**Phase 1 (MVP - Current Requirements)**:
- Hindi and Marathi support as specified in requirements
- Core functionality within AWS free tier

**Phase 2 (Language Expansion)**:
- Add support for 9 additional Indian languages:
  - Tamil, Telugu, Bengali (major South and East Indian languages)
  - Kannada, Malayalam (South Indian languages)
  - Gujarati, Punjabi (West and North Indian languages)
  - Odia, Assamese (East Indian languages)
- Requires minimal code changes (architecture already supports multi-language)
- Main effort: Translating scheme documents and testing

### Phase 2 Features (Within Free Tier)

1. **Enhanced Caching**:
   - Implement predictive cache warming based on usage patterns
   - Machine learning for cache eviction policies
   - Distributed cache synchronization

2. **Improved Summarization**:
   - Better rule-based extractive summarization
   - Template-based summaries for common scheme types
   - Keyword-based sentence ranking algorithms

3. **User Experience**:
   - Progressive response delivery (stream audio as it's generated)
   - Offline mode with pre-cached popular schemes
   - SMS-based interface for feature phones (no audio processing)

4. **Analytics**:
   - User behavior analytics (within DynamoDB free tier)
   - Popular query tracking for cache optimization
   - Usage pattern analysis for better resource allocation

### Phase 3 Features (Requires Paid Tier)

1. **Amazon Bedrock Integration**:
   - Upgrade from Amazon Comprehend to Amazon Bedrock as specified in Requirement 4.1
   - Use Claude or Titan for higher-quality abstractive summarization
   - Implement conversational AI for follow-up questions
   - Generate personalized recommendations
   - Multi-turn dialogue support

2. **Additional Languages Beyond Phase 2**:
   2. **Additional Languages Beyond Phase 2**:
   - Support for more Indian regional languages beyond the 11 planned
   - Automatic language detection without user specification
   - Dialect support within languages

3. **Personalization**:
   3. **Personalization**:
   - User profiles with saved preferences
   - Personalized scheme recommendations based on demographics
   - Conversation history for context-aware responses

4. **Advanced Search**:
   4. **Advanced Search**:
   - Semantic search using vector embeddings
   - Multi-turn conversations for clarification
   - Follow-up questions without repeating context

5. **Multi-Modal Interface**:
   5. **Multi-Modal Interface**:
   - Support for text input in addition to voice
   - Visual display of scheme information with images
   - Video tutorials for scheme applications

6. **Better AI** (Upgrade from Comprehend to Bedrock):
   6. **Better AI** (Upgrade from Comprehend to Bedrock):
   - Fine-tuned models on government scheme data
   - Use Amazon Bedrock for better summarization quality (as specified in Requirement 4.1)
   - Implement fact-checking to prevent hallucinations

### Infrastructure Enhancements (Paid Tier)

1. **Performance**:
   - Implement ElastiCache Redis for distributed caching
   - Use DynamoDB DAX for microsecond latency
   - Enable Lambda provisioned concurrency
   - Implement response streaming for faster perceived performance

2. **Monitoring**:
   - Real-time dashboards for system health
   - User analytics and usage patterns
   - A/B testing framework for feature improvements
   - Advanced CloudWatch metrics and alarms

3. **Scalability**:
   - Remove user throttling limits
   - Support thousands of concurrent users
   - Multi-region deployment for global access
   - CDN integration for faster content delivery

4. **Security**:
   - Advanced fraud detection for abuse prevention
   - Implement data residency controls for compliance
   - Enhanced encryption with customer-managed keys
   - Regular security audits and penetration testing

### Alternative Architecture (Post-Free-Tier)

**Open Source Alternative Stack**:
- Replace AWS Transcribe with Whisper (OpenAI) or Vosk
- Replace AWS Polly with Mozilla TTS or Coqui TTS
- Replace AWS Translate with MarianMT or NLLB
- Replace Amazon Comprehend with spaCy or NLTK
- Host on AWS Lambda (still free tier) or migrate to other platforms

**Estimated Cost Savings**:
- Eliminate Transcribe cost ($0.024/minute)
- Eliminate Translate cost ($15/million characters)
- Eliminate Comprehend cost ($0.0001/unit)
- Trade-off: Increased Lambda compute time, lower quality

### Voice Quality Enhancements (Paid Tier)

1. **Better TTS**:
   - Use AWS Polly neural voices (not in free tier)
   - Implement emotion and emphasis in TTS
   - Support for different speaking speeds
   - Custom voice training for regional accents

2. **Better STT**:
   - Use AWS Transcribe custom vocabulary
   - Train on government scheme terminology
   - Support for noisy environments
   - Real-time streaming transcription

### AI Enhancements (Paid Tier)

1. **Amazon Bedrock Integration** (Fulfills Requirement 4.1):
   - Upgrade from Amazon Comprehend to Amazon Bedrock
   - Use Claude or Titan for better summarization
   - Implement conversational AI for follow-up questions
   - Generate personalized recommendations
   - Multi-turn dialogue support

2. **Understanding**:
   - Implement intent classification for better query understanding
   - Add entity extraction for scheme names and categories
   - Support for colloquial language and dialects
   - Context-aware responses based on user history
