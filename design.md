# Design Document: AI Skill-Gap Based Personalized Project Generator

## Overview

A MERN stack platform that analyzes developer skills through GitHub data, identifies skill gaps, and generates personalized project recommendations with starter code. The system uses LLMs for intelligent generation and continuously updates skill assessments based on user progress.

## Architecture

### System Architecture
graph TB
    User[Developer]
    Frontend[React Frontend]
    API[Express API Gateway]

    subgraph BackendServices
        Auth[Auth Service]
        GitHubSvc[GitHub Service]
        Analysis[Code Analysis Service]
        Skills[Skill Graph Service]
        Gaps[Skill Gap Service]
        Projects[Project Generation Service]
        Tasks[Task Breakdown Service]
        Templates[Code Template Service]
        Progress[Progress Tracker]
        Learning[Learning Service]
    end

    subgraph ExternalAPIs
        GitHubAPI[GitHub API]
        LLM[LLM APIs]
        Embeddings[Embedding Service]
    end

    subgraph DataLayer
        MongoDB[(MongoDB)]
        Redis[(Redis Cache)]
        VectorDB[(Vector Store)]
    end

    Queue[Job Queue]

    User --> Frontend
    Frontend --> API

    API --> Auth
    API --> GitHubSvc
    API --> Analysis
    API --> Skills
    API --> Gaps
    API --> Projects
    API --> Tasks
    API --> Templates
    API --> Progress
    API --> Learning

    GitHubSvc --> GitHubAPI
    Analysis --> LLM
    Projects --> LLM
    Learning --> Embeddings

    Auth --> MongoDB
    Skills --> VectorDB
    Gaps --> MongoDB
    Progress --> Redis

    Projects --> Queue
    Tasks --> Queue


### Technology Stack

**Frontend**: React 18, React Router, Axios, Recharts, Tailwind CSS, React Query

**Backend**: Node.js 18+, Express.js 4.x, Mongoose, Passport.js (JWT + GitHub OAuth), Bull/BullMQ, Winston

**Database**: MongoDB 6.0+ (primary data), Redis (caching/sessions), Chroma/FAISS (vector embeddings)

**AI/ML**: OpenAI GPT-4, Anthropic Claude, DeepSeek, Ollama (Llama), InstructorXL/OpenAI Embeddings

**DevOps**: Docker, Docker Compose, GitHub Actions, PM2, Nginx

## Modules

### 1. Authentication Service
Handles user registration, login, JWT token management, and GitHub OAuth integration. Encrypts sensitive data (passwords, GitHub tokens) before storage.

### 2. GitHub Service
Fetches user repositories, commit history, and metadata via GitHub API. Detects programming languages, frameworks from dependency files, and calculates commit metrics (frequency, consistency).

### 3. Code Analysis Service
Performs static analysis on code files to calculate complexity metrics, detect bug patterns, identify architecture patterns, assess testing practices, and generate code embeddings for similarity search.

### 4. Skill Graph Service
Aggregates analysis results into skill profiles with scores (0-100) across seven categories: Languages, Frameworks, Architecture, Testing, DevOps, Algorithms, Databases. Applies recency weighting to prioritize recent activity. Maintains version history for progress tracking.

### 5. Skill Gap Service
Compares current skill graph against target career path profiles. Identifies weak skills (score < 60) and missing skills (score = 0). Prioritizes gaps by importance, urgency, and learning difficulty. Generates actionable skill gap reports with learning resource recommendations.

### 6. Project Generation Service
Uses LLM to generate personalized project ideas based on skill gaps and user preferences. Ensures projects address 2-4 skill gaps, match time availability, align with preferred tech stack, and include core features, stretch goals, and learning outcomes.

### 7. Task Breakdown Service
Converts project ideas into 5-20 actionable tasks using LLM. Assigns difficulty levels (Easy/Medium/Hard), time estimates, learning outcomes, and identifies task dependencies. Calculates critical path and total project duration.

### 8. Code Template Service
Generates project scaffolding including file structure, boilerplate code, configuration files (package.json, .env), README with setup instructions, and test scaffolding. Packages output as downloadable ZIP file.

### 9. Progress Tracker
Monitors GitHub repositories for new commits (hourly sync). Analyzes new code and updates skill graphs. Triggers project recommendations when significant progress detected. Enforces rate limiting to avoid GitHub API throttling.

### 10. Learning Service
Generates daily micro-lessons (5-10 minutes) targeting skill gaps. Creates flashcards for key concepts. Provides concept summaries with examples. Tracks lesson completion and suggests next steps.

## Data Flow

### Initial Analysis Flow
1. User authenticates via GitHub OAuth
2. System fetches repositories and commit history
3. Code analysis generates embeddings and metrics
4. Skill graph is calculated and stored with version number
5. Skill gaps are identified against target career path
6. Results displayed in dashboard with visualizations

### Project Generation Flow
1. User requests project recommendation
2. System retrieves skill graph and gap report from MongoDB
3. LLM generates personalized project idea
4. Task breakdown is created with dependencies
5. Code template is generated and cached
6. User downloads template as ZIP file

### Progress Tracking Flow
1. Background job polls GitHub API hourly
2. New commits are detected and analyzed
3. Skill graph is updated (new version created)
4. Skill gaps are recalculated
5. Notifications sent for improvements
6. New project recommendations triggered if significant progress

## Data Models

### User
Stores authentication credentials, GitHub connection, learning preferences (goals, time availability, tech stack, difficulty), and account metadata.

### SkillGraph
Contains versioned skill profiles with scores per category, confidence levels, evidence counts, timestamps, and overall proficiency level (Beginner/Intermediate/Advanced).

### SkillGap
Stores identified gaps categorized by priority (critical/high/medium/low), current vs target scores, estimated learning time, and recommended resources.

### Project
Contains generated project details: title, objective, description, required skills, technologies, skill gaps addressed, difficulty, features, stretch goals, learning outcomes, justification, and status.

### Task
Stores task breakdowns: title, description, difficulty, estimated hours, order, prerequisites, learning outcomes, acceptance criteria, and completion status.

### LearningProgress
Tracks completed lessons, flashcards, and exercises with scores and time spent.

## Correctness Properties

*Properties are universal characteristics that should hold true across all valid executions.*

### Key Properties

**Authentication**: Passwords must be encrypted; JWT tokens valid for 24 hours; GitHub tokens encrypted at rest.

**Skill Graphs**: All scores in range 0-100; all seven categories present; recent activity weighted higher; version history preserved.

**Skill Gaps**: Weak skills have score < 60; missing skills have score = 0; gaps prioritized by importance and urgency.

**Projects**: Address 2-4 skill gaps; include all required fields; scope matches time availability; tech stack aligned with preferences.

**Tasks**: Count between 5-20; all have difficulty, time estimate, learning outcomes; dependencies respected in ordering; total duration equals sum of task hours.

**Templates**: Include file structure, boilerplate, config files, README, .env.example; output is valid ZIP file.

**Progress**: GitHub sync at most once per hour; new commits trigger skill graph updates; skill gaps recalculated on updates.

**Security**: JWT authentication enforced; rate limiting at 100 req/min; HTTPS for all transmissions; sensitive data encrypted.

## Error Handling

### Strategies

**Retry with Exponential Backoff**: For transient failures (GitHub API, LLM API, network issues)

**Graceful Degradation**: Use cached data if GitHub API fails; template-based fallbacks if LLM fails; simpler analysis if embedding fails

**Error Logging**: Sanitize sensitive data; log with context and severity; send to monitoring service

**User-Facing Messages**: Clear, actionable errors; no internal details exposed; include remediation steps

### Error Categories

- Authentication errors (invalid credentials, expired tokens)
- External API errors (rate limits, timeouts, unavailability)
- Validation errors (invalid input, missing fields, out-of-range values)
- Database errors (connection failures, query timeouts)
- Business logic errors (insufficient data, invalid state)

## Testing Strategy

### Dual Approach

**Unit Tests**: Specific examples, edge cases, integration points using Mocha/Chai/Sinon

**Property Tests**: Universal properties across randomized inputs using fast-check (100+ iterations per property)

### Coverage Requirements

- Overall: 80% minimum
- Core business logic: 90% minimum
- Security-critical code: 100% required
- API endpoints: 95% minimum

### Test Types

- Unit tests for individual services and functions
- Property tests for correctness properties
- Integration tests for end-to-end flows
- API tests for endpoint behavior
- Load tests for performance validation
- Security tests for vulnerability scanning

## Constraints

### Technical Constraints

- GitHub API rate limit: 5000 requests/hour (authenticated)
- LLM API costs and latency (2-5 seconds per generation)
- MongoDB document size limit: 16MB
- Code embedding dimension: 1536 (OpenAI) or 768 (InstructorXL)
- JWT token expiry: 24 hours
- Rate limiting: 100 requests/minute per user

### Business Constraints

- Users must have active GitHub profiles
- Analysis requires at least one public repository or uploaded code
- Project generation requires completed skill analysis
- Free tier limits: 5 projects/month, 10 analyses/month
- Premium tier: unlimited projects and analyses

### Performance Constraints

- API response time: < 300ms for standard calls
- Analysis completion: < 5 minutes for typical repository
- Project generation: < 10 seconds
- Task breakdown: < 5 seconds
- Template generation: < 3 seconds
- GitHub sync interval: minimum 60 minutes
