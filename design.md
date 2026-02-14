# CodeMentor AI - Design Document

## Architecture Overview

CodeMentor AI follows a modern cloud-native architecture built on AWS infrastructure, utilizing a microservices approach for scalability and maintainability. The system consists of three main layers:

1. **Presentation Layer**: React-based SPA providing responsive UI
2. **Application Layer**: Microservices handling business logic (Node.js/FastAPI)
3. **Data Layer**: DynamoDB for persistence, S3 for static assets, CloudWatch for monitoring

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         React SPA (Hosted on S3 + CloudFront)        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway (AWS)                         │
│              Authentication & Rate Limiting                  │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   User       │   │   AI Tutor   │   │   Project    │
│   Service    │   │   Service    │   │   Service    │
│  (Node.js)   │   │  (FastAPI)   │   │  (Node.js)   │
└──────────────┘   └──────────────┘   └──────────────┘
        │                   │                   │
        │                   ▼                   │
        │          ┌──────────────┐            │
        │          │ AWS Bedrock  │            │
        │          │   (Claude)   │            │
        │          └──────────────┘            │
        │                                       │
        └───────────────────┬───────────────────┘
                            ▼
                   ┌──────────────┐
                   │  DynamoDB    │
                   │   Tables     │
                   └──────────────┘
```

## System Components

### 1. Frontend Application (React SPA)

**Purpose**: Provide intuitive, responsive user interface for all platform features

**Responsibilities**:
- User authentication and session management
- Dashboard rendering (streaks, study hours, progress)
- AI tutor chat interface
- Project display and management
- Pomodoro timer UI and notifications
- Progress visualization and analytics

**Technology Stack**:
- React 18+ with hooks
- React Router for navigation
- Redux Toolkit for state management
- Axios for API communication
- Material-UI or Tailwind CSS for styling
- Chart.js for data visualization
- React Query for server state management

**Key Modules**:
```
src/
├── components/
│   ├── auth/
│   ├── dashboard/
│   ├── tutor/
│   ├── projects/
│   ├── pomodoro/
│   └── common/
├── services/
│   ├── api.js
│   ├── auth.js
│   └── websocket.js
├── store/
│   ├── slices/
│   └── store.js
└── utils/
```

### 2. API Gateway

**Purpose**: Single entry point for all client requests, handles routing, authentication, and rate limiting

**Responsibilities**:
- Request routing to appropriate microservices
- JWT token validation
- Rate limiting and throttling
- CORS handling
- Request/response transformation
- API versioning

**Configuration**:
- AWS API Gateway (REST API)
- Lambda authorizer for JWT validation
- Usage plans for rate limiting
- CloudWatch integration for monitoring

### 3. User Service (Node.js)

**Purpose**: Manage user accounts, authentication, profiles, and preferences

**Responsibilities**:
- User registration and login
- Password hashing and validation (bcrypt)
- JWT token generation and refresh
- OAuth integration (Google, GitHub)
- Profile management
- User preferences and settings
- Session tracking

**API Endpoints**:
```
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
GET    /api/v1/users/profile
PUT    /api/v1/users/profile
GET    /api/v1/users/preferences
PUT    /api/v1/users/preferences
```

**Technology**:
- Express.js framework
- JWT for authentication
- bcrypt for password hashing
- AWS Lambda + API Gateway (serverless)

### 4. AI Tutor Service (FastAPI)

**Purpose**: Handle all AI-powered tutoring interactions using Claude via AWS Bedrock

**Responsibilities**:
- Process student questions
- Generate Socratic responses
- Code analysis and debugging
- Context management for conversations
- Skill-level adaptation
- Conversation history management
- Response streaming for real-time feel

**API Endpoints**:
```
POST   /api/v1/tutor/ask
GET    /api/v1/tutor/history/:userId
POST   /api/v1/tutor/analyze-code
GET    /api/v1/tutor/conversation/:conversationId
DELETE /api/v1/tutor/conversation/:conversationId
```

**Technology**:
- FastAPI framework (Python)
- AWS Bedrock SDK for Claude integration
- Prompt engineering templates
- Context window management
- Response caching (Redis/ElastiCache)

**AI Integration Flow**:
```
Student Question
      ↓
Context Retrieval (user level, history)
      ↓
Prompt Construction (Socratic method)
      ↓
AWS Bedrock (Claude API)
      ↓
Response Processing
      ↓
Store in History
      ↓
Return to Client
```

### 5. Project Service (Node.js)

**Purpose**: Generate, manage, and track coding projects tailored to student needs

**Responsibilities**:
- Generate custom projects based on skill level
- Create project milestones
- Track project progress
- Suggest next projects
- Provide starter code and templates
- Project difficulty assessment

**API Endpoints**:
```
POST   /api/v1/projects/generate
GET    /api/v1/projects/:userId
GET    /api/v1/projects/:projectId
PUT    /api/v1/projects/:projectId/progress
POST   /api/v1/projects/:projectId/complete
GET    /api/v1/projects/suggestions/:userId
```

**Project Generation Logic**:
- Input: User skill level, interests, completed projects
- Process: Use Claude to generate project specifications
- Output: Project description, milestones, starter code, resources

### 6. Progress Service (Node.js)

**Purpose**: Track and analyze student learning progress, study habits, and achievements

**Responsibilities**:
- Pomodoro session tracking
- Study streak calculation
- Total hours tracking
- Achievement badge management
- Progress analytics
- Learning path recommendations

**API Endpoints**:
```
POST   /api/v1/progress/session/start
POST   /api/v1/progress/session/end
GET    /api/v1/progress/stats/:userId
GET    /api/v1/progress/streaks/:userId
GET    /api/v1/progress/achievements/:userId
GET    /api/v1/progress/recommendations/:userId
```

## Data Model

### DynamoDB Tables

#### Users Table
```
PK: USER#<userId>
SK: PROFILE

Attributes:
- userId (String, UUID)
- email (String)
- passwordHash (String)
- firstName (String)
- lastName (String)
- skillLevel (String: beginner|intermediate|advanced)
- goals (List<String>)
- interests (List<String>)
- createdAt (Number, timestamp)
- updatedAt (Number, timestamp)
- lastLoginAt (Number, timestamp)

GSI1: email-index (for login lookup)
```

#### Conversations Table
```
PK: USER#<userId>
SK: CONV#<conversationId>

Attributes:
- conversationId (String, UUID)
- userId (String)
- messages (List<Map>)
  - role (String: user|assistant)
  - content (String)
  - timestamp (Number)
- createdAt (Number)
- updatedAt (Number)

GSI1: conversationId-index
```

#### Projects Table
```
PK: USER#<userId>
SK: PROJECT#<projectId>

Attributes:
- projectId (String, UUID)
- userId (String)
- title (String)
- description (String)
- difficulty (String)
- skillLevel (String)
- milestones (List<Map>)
  - id (String)
  - title (String)
  - description (String)
  - completed (Boolean)
  - completedAt (Number)
- status (String: not_started|in_progress|completed)
- starterCode (String)
- resources (List<String>)
- createdAt (Number)
- completedAt (Number)

GSI1: status-index
```

#### StudySessions Table
```
PK: USER#<userId>
SK: SESSION#<timestamp>

Attributes:
- sessionId (String, UUID)
- userId (String)
- startTime (Number)
- endTime (Number)
- duration (Number, minutes)
- sessionType (String: pomodoro|custom)
- focusScore (Number, 1-10)
- notes (String)

GSI1: date-index (for daily/weekly queries)
```

#### Achievements Table
```
PK: USER#<userId>
SK: ACHIEVEMENT#<achievementId>

Attributes:
- achievementId (String)
- userId (String)
- type (String: streak|hours|projects|questions)
- title (String)
- description (String)
- unlockedAt (Number)
- metadata (Map)
```

### Data Access Patterns

1. Get user profile: Query by PK=USER#<userId>, SK=PROFILE
2. Get user conversations: Query by PK=USER#<userId>, SK begins_with CONV#
3. Get user projects: Query by PK=USER#<userId>, SK begins_with PROJECT#
4. Get study sessions by date range: Query GSI1 with date range
5. Login lookup: Query GSI1 (email-index) by email

## API Design

### Authentication Flow

```
1. User Registration:
   POST /api/v1/auth/register
   Body: { email, password, firstName, lastName, skillLevel }
   Response: { userId, accessToken, refreshToken }

2. User Login:
   POST /api/v1/auth/login
   Body: { email, password }
   Response: { userId, accessToken, refreshToken }

3. Token Refresh:
   POST /api/v1/auth/refresh
   Body: { refreshToken }
   Response: { accessToken }

4. Protected Endpoints:
   Header: Authorization: Bearer <accessToken>
```

### AI Tutor Interaction Flow

```
1. Ask Question:
   POST /api/v1/tutor/ask
   Headers: { Authorization: Bearer <token> }
   Body: {
     question: "How do I use async/await?",
     context: {
       skillLevel: "beginner",
       currentTopic: "JavaScript Promises"
     }
   }
   Response: {
     conversationId: "uuid",
     response: "Great question! Let me guide you...",
     followUpQuestions: ["What do you know about Promises?"]
   }

2. Get Conversation History:
   GET /api/v1/tutor/history/:userId?limit=20&offset=0
   Response: {
     conversations: [...]
   }
```

### Project Generation Flow

```
1. Generate Project:
   POST /api/v1/projects/generate
   Body: {
     skillLevel: "intermediate",
     interests: ["web development", "APIs"],
     previousProjects: ["todo-app", "weather-app"]
   }
   Response: {
     projectId: "uuid",
     title: "Recipe Finder API",
     description: "Build a REST API...",
     milestones: [...],
     estimatedHours: 8
   }

2. Update Progress:
   PUT /api/v1/projects/:projectId/progress
   Body: {
     milestoneId: "milestone-1",
     completed: true
   }
```

### Progress Tracking Flow

```
1. Start Pomodoro Session:
   POST /api/v1/progress/session/start
   Body: { sessionType: "pomodoro", duration: 25 }
   Response: { sessionId: "uuid", startTime: 1234567890 }

2. End Session:
   POST /api/v1/progress/session/end
   Body: { sessionId: "uuid", focusScore: 8 }
   Response: { 
     duration: 25,
     streakUpdated: true,
     currentStreak: 7
   }

3. Get Stats:
   GET /api/v1/progress/stats/:userId
   Response: {
     totalHours: 45.5,
     currentStreak: 7,
     longestStreak: 14,
     projectsCompleted: 5,
     questionsAsked: 127
   }
```

## Security Design

### Authentication & Authorization

- JWT-based authentication with access and refresh tokens
- Access token expiry: 15 minutes
- Refresh token expiry: 7 days
- Tokens stored in httpOnly cookies (web) or secure storage (mobile)
- Role-based access control (RBAC) for future admin features

### Data Protection

- Passwords hashed with bcrypt (salt rounds: 12)
- Sensitive data encrypted at rest (DynamoDB encryption)
- TLS 1.3 for data in transit
- API Gateway with AWS WAF for DDoS protection
- Input validation and sanitization on all endpoints
- Rate limiting: 100 requests/minute per user

### Compliance

- GDPR: User data export and deletion endpoints
- COPPA: Age verification for users under 13
- Privacy policy and terms of service acceptance tracking
- Audit logging for sensitive operations

## Deployment Architecture

### AWS Infrastructure

```
┌─────────────────────────────────────────────────────────┐
│                    Route 53 (DNS)                        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              CloudFront (CDN + SSL)                      │
└─────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┴─────────────────┐
        ▼                                   ▼
┌──────────────┐                   ┌──────────────┐
│   S3 Bucket  │                   │ API Gateway  │
│  (Frontend)  │                   │   (REST)     │
└──────────────┘                   └──────────────┘
                                           │
                    ┌──────────────────────┼──────────────┐
                    ▼                      ▼              ▼
            ┌──────────────┐      ┌──────────────┐  ┌─────────┐
            │   Lambda     │      │   Lambda     │  │ Lambda  │
            │ (User Svc)   │      │ (Tutor Svc)  │  │ (Proj)  │
            └──────────────┘      └──────────────┘  └─────────┘
                    │                      │              │
                    └──────────────────────┼──────────────┘
                                           ▼
                                  ┌──────────────┐
                                  │  DynamoDB    │
                                  └──────────────┘
```

### CI/CD Pipeline

```
GitHub Repository
      ↓
GitHub Actions (on push to main)
      ↓
1. Run Tests (Jest, Pytest)
      ↓
2. Build Frontend (npm run build)
      ↓
3. Build Backend (Docker images)
      ↓
4. Deploy to Staging
      ↓
5. Run Integration Tests
      ↓
6. Deploy to Production (manual approval)
      ↓
7. Health Checks
```

### Environment Configuration

- Development: Local development with LocalStack for AWS services
- Staging: AWS environment mirroring production
- Production: Full AWS infrastructure with auto-scaling

## Monitoring & Observability

### Metrics (CloudWatch)

- API response times (p50, p95, p99)
- Error rates by endpoint
- Lambda invocation counts and durations
- DynamoDB read/write capacity usage
- Bedrock API latency and token usage
- Active user sessions

### Logging

- Structured JSON logging
- CloudWatch Logs for centralized log aggregation
- Log levels: ERROR, WARN, INFO, DEBUG
- Request/response logging with correlation IDs
- Error stack traces and context

### Alerting

- API error rate > 5%: Page on-call engineer
- Response time p95 > 5s: Send alert
- Lambda errors: Slack notification
- DynamoDB throttling: Auto-scale trigger
- Bedrock API failures: Fallback to cached responses

## Performance Optimization

### Caching Strategy

- CloudFront: Static assets (1 year TTL)
- API Gateway: GET responses (5 minutes TTL)
- ElastiCache (Redis): 
  - User sessions (15 minutes)
  - Conversation context (1 hour)
  - Project templates (24 hours)

### Database Optimization

- DynamoDB on-demand pricing for variable load
- GSI for efficient query patterns
- Batch operations for bulk reads/writes
- Connection pooling for Lambda functions

### Frontend Optimization

- Code splitting by route
- Lazy loading for heavy components
- Image optimization and lazy loading
- Service worker for offline capability
- Debouncing for search and input fields

## Scalability Considerations

### Horizontal Scaling

- Lambda auto-scales based on request volume
- DynamoDB auto-scales read/write capacity
- CloudFront handles global distribution
- API Gateway handles millions of requests

### Cost Optimization

- Lambda: Pay per invocation
- DynamoDB: On-demand pricing
- S3: Lifecycle policies for old data
- CloudWatch: Log retention policies (30 days)
- Bedrock: Token usage monitoring and limits

## Implementation Phases

### Phase 1: MVP (Months 1-2)
- User authentication and profiles
- Basic AI tutor with Claude integration
- Simple project generator
- Pomodoro timer
- Basic dashboard

### Phase 2: Enhanced Features (Month 3)
- Advanced progress tracking
- Achievement system
- Project milestone tracking
- Conversation history
- Analytics dashboard

### Phase 3: Polish & Optimization (Month 4)
- Performance optimization
- Security hardening
- Comprehensive testing
- Documentation
- Pilot program preparation

### Phase 4: Pilot Program (Months 5-10)
- User onboarding
- Feedback collection
- Iterative improvements
- Scale testing
- Production readiness

## Technology Stack Summary

### Frontend
- React 18+
- Redux Toolkit
- React Router
- Material-UI / Tailwind CSS
- Axios / React Query
- Chart.js

### Backend
- Node.js (Express) for User & Project services
- Python (FastAPI) for AI Tutor service
- AWS Lambda (serverless)
- AWS API Gateway

### AI & ML
- AWS Bedrock (Claude)
- Prompt engineering
- Context management

### Database & Storage
- DynamoDB (NoSQL)
- S3 (static assets)
- ElastiCache (Redis) for caching

### Infrastructure
- AWS CloudFront (CDN)
- Route 53 (DNS)
- CloudWatch (monitoring)
- AWS WAF (security)
- AWS Secrets Manager

### DevOps
- GitHub Actions (CI/CD)
- Docker (containerization)
- AWS CloudFormation / Terraform (IaC)
- Jest / Pytest (testing)

## Risk Mitigation

### Technical Risks
- Bedrock API downtime: Implement fallback responses and caching
- High AWS costs: Set billing alerts and usage limits
- Performance issues: Load testing and optimization before launch
- Data loss: Automated backups and point-in-time recovery

### Business Risks
- Low user adoption: Pilot program for validation
- Competition: Focus on unique pedagogy-first approach
- Scalability: Start small, scale based on demand

## Future Enhancements

- Real-time collaboration features (WebSocket)
- Mobile native apps (React Native)
- Code execution environment (sandboxed IDE)
- Video tutorials and content library
- Instructor dashboard for educators
- Advanced analytics with ML insights
- Integration with external platforms (GitHub, LeetCode)
- Community features (forums, peer review)
