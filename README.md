# COSMOS Intelligent Tutoring System (ITS)

> A comprehensive AI-powered educational platform with a multi-agent RAG chatbot, ML-based grade prediction, adaptive learning roadmaps, academic performance tracking, and a full-featured admin panel — all orchestrated through a modular monorepo architecture.

COSMOS-ITS is an end-to-end intelligent tutoring system designed for university-level education. It combines a **FastAPI microservice backend** (`cosmos-its-server`) with a **Next.js admin and student portal** (`cosmos-admin-panel`) to deliver a seamless experience for both administrators and students.

---

## System Architecture

```
cosmos-its/
├── cosmos-its-server/          # FastAPI + LangChain + LangGraph backend (Python)
│   ├── api/                    # Vercel serverless entry point
│   ├── src/                    # Application source
│   │   ├── core/               # Infrastructure, security, agent framework
│   │   ├── features/           # Domain modules (auth, chat, prediction, roadmap, etc.)
│   │   ├── config/             # Settings & constants
│   │   └── main.py             # FastAPI app bootstrap
│   ├── models/                 # Trained ML model artifacts (.joblib)
│   ├── requirements.txt        # Python dependencies
│   └── vercel.json             # Serverless deployment config
│
├── cosmos-admin-panel/         # Next.js admin dashboard + student portal (TypeScript)
│   ├── app/                    # App Router pages & API routes
│   │   ├── admin/              # Admin dashboard, CRUD modules, logs, search
│   │   ├── user/               # Student portal (chat, performance, roadmap, CGPA)
│   │   ├── chat/               # AI-powered chat interface
│   │   ├── login/              # Authentication pages
│   │   └── api/                # Next.js API routes (auth, CRUD, search, embeddings)
│   ├── components/             # shadcn/ui + custom components
│   ├── lib/                    # Core services (auth, MongoDB, Pinecone, OpenAI)
│   ├── store/                  # Zustand state management
│   ├── hooks/                  # Custom React hooks
│   ├── middleware.ts           # Edge runtime JWT validation & RBAC
│   └── server.js               # cPanel Phusion Passenger deployment
│
└── README.md                   # This file
```

### High-Level Data Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Admin Panel (Next.js)                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐  │
│  │ Dashboard│  │ Users/   │  │ Course   │  │ AI Agent Config    │  │
│  │ Analytics│  │ Students │  │ Mgmt     │  │ & Tools            │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐  │
│  │ Question │  │ Search   │  │ System   │  │ Embeddings /       │  │
│  │ Bank     │  │ (Global) │  │ Logs     │  │ Vector Management  │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────────┘  │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ HTTP / REST
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     ITS Server (FastAPI)                              │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │           LangGraph Orchestrator (Dynamic Agent Router)      │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────────────┐ │   │
│  │  │Supervisor│  │ Course  │  │ General │  │ Roadmap Agent  │ │   │
│  │  │  Agent   │  │  Agent  │  │  Agent  │  │                │ │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  Pinecone    │  │    OpenAI    │  │  ML Ensemble (RF+GB+MLP) │  │
│  │  Vector DB   │  │    LLM       │  │  Grade Prediction        │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              MongoDB (Beanie ODM / Motor)                     │   │
│  │  Accounts │ Threads │ Agents │ Courses │ Assessments │ ...   │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Features

### Multi-Agent RAG Chatbot

The system employs a **dynamic multi-agent architecture** built on LangGraph. Rather than a single monolithic AI, the orchestrator routes queries to specialized agents based on context:

| Agent | Role |
|---|---|
| **Supervisor Agent** | Context-aware router — assigns highest priority to pagination requests, routes to course-specific agents or the general fallback |
| **Course Agents** | Subject-matter experts for courses like DBMS, SPL, OOP, etc. Each agent performs intent detection (conceptual question vs. retrieval request), contextual query rewriting, and structured response generation |
| **General Agent** | Fallback for general education queries outside specific course scope |

**RAG Pipeline**: User query → Supervisor routing → Course agent (intent detection) → Query rewriting with conversation context → LLM classification using few-shot learning from MongoDB → Pinecone vector search with metadata filters → Structured results with pagination state tracking.

Agents are **auto-discovered** — creating a new agent requires only a Python file extending `BaseAgent` under the features directory and a MongoDB record. No hardcoded imports needed.

### ML-Based Grade Prediction

A production-grade ensemble prediction system that forecasts student grades:

- **Models**: RandomForest, GradientBoosting, and MLP neural network trained on historical assessment data
- **Ensemble**: Weighted voting across all three models for robust predictions
- **Meta-Learning**: Adjusts predictions based on student experience level (fresh vs. experienced) and cross-course learning patterns
- **Feature Engineering**: Course clusters, CT (class test) trends, trimester code resolution, marking scheme application (theory/lab distinction)
- **Storage**: Pre-trained joblib artifacts for fast inference

### Adaptive Learning Roadmaps

AI-generated personalized study roadmaps with multi-graph architecture:

- **Generation Graph**: Structured roadmap creation (stages, items, descriptions) via LLM with JSON schema validation
- **Chat Graph**: Context-aware follow-up Q&A about roadmap content with node-specific context injection
- **Explanation Graph**: Deep-dive topic explanations anchored to specific roadmap nodes
- **Progress Tracking**: Node-level completion tracking with toggle functionality
- **Checkpointing**: LangGraph MongoDB checkpointer for conversation persistence

### Academic Performance Tracker

Comprehensive student performance management:

- **Assessments**: Full CRUD with grade calculation per the institution's marking scheme
- **Topics**: Course topic tracking with per-student confidence levels
- **Quiz Sessions**: Quiz generation, submission, and grading
- **Weakness Analysis**: Track and analyze student weak areas
- **GPA Calculation**: Full GPA computation with trimester-level records, grade scale enforcement (A=4.0 through F=0.0), and course-by-course breakdowns
- **Course Enrollment**: Student-course mapping with marks and grades

### Student Calendar & Events

- CRUD operations for personal academic events with date-range filtering
- Deduplication logic for event management
- Integration with the academic calendar system

### Admin Panel Features

The admin panel (`cosmos-admin-panel`) provides a complete management interface:

| Module | Capabilities |
|---|---|
| **Dashboard Analytics** | Real-time stats (users, courses, questions, agents), system operations, success rates, bar/pie charts, activity feed |
| **User Management** | CRUD with search/filter, role-based access (admin/student), auto-generated secure passwords, email notifications, profile editing, academic data management |
| **Question Bank** | Full CRUD with course/exam-type/semester filtering, rich text editing, image support, vector embedding generation, Pinecone sync |
| **Course Management** | Course, topic, and trimester CRUD with department tracking, exam type categorization |
| **AI Agent Management** | Create/edit/activate/deactivate agents, configure system prompts, tool bindings, few-shot examples, and extended configurations |
| **Embedding Management** | Update/reprocess vector embeddings per course with shimmer-animated progress tracking and upsert statistics |
| **Content Delivery** | CDN file browser with upload, type filtering (pdf/image), paginated grid, download links |
| **System Logs** | Full immutable audit trail with admin actions, request metadata (IP, user-agent, duration), searchable by method/resource/date range |
| **Global Search** | Cross-collection command palette (cmd+k) searching users, questions, courses, agents, system logs, and navigation pages |
| **Help Center** | Searchable help articles with relevance scoring, accordion FAQs, command palette, support contact |

### Student Portal Features

| Module | Capabilities |
|---|---|
| **Student Dashboard** | Enrolled courses, performance summaries, CGPA display, GPA trend radar chart, notices, upcoming events |
| **Performance Module** | Course details with tabs (Assessments, Quizzes, Reports), quiz taking with history, weakness analysis, grade prediction |
| **Learning Roadmap** | Interactive D3 force-directed graph visualization with zoom/pan and detailed panels |
| **CGPA Calculator** | Interactive GPA calculator with retake support |
| **Class Routines** | Class schedule + exam timetable with UIU profile integration |
| **Study Planner** | Calendar-based event planner with CRUD dialogs |
| **Notices & Academic Calendar** | Paginated notice list with detail view, calendar entries with revised indicators, PDF downloads |
| **Profile Settings** | Profile editor and password change |
| **AI Chat** | Full-featured multi-agent chat with streaming responses, thread management, markdown rendering, code block highlighting |

---

## Technology Stack

### Backend (`cosmos-its-server`)

| Category | Technologies |
|---|---|
| **Framework** | Python 3.11, FastAPI, Uvicorn, Pydantic |
| **AI/LLM** | OpenAI GPT-4/GPT-3.5, LangChain, LangGraph, LangChain-Core, LangChain-Community |
| **Vector Database** | Pinecone (OpenAI text-embedding-3-small / ada-002) |
| **Database** | MongoDB via Motor + Beanie ODM + PyMongo |
| **Auth** | PyJWT, bcrypt (12 rounds), Passlib |
| **ML** | scikit-learn (RandomForest, GradientBoosting), NumPy, Pandas, Joblib |
| **Orchestration** | LangGraph StateGraph with MongoDB checkpointing |
| **Deployment** | Vercel Serverless (Python 3.11, 250MB lambda) |

### Frontend (`cosmos-admin-panel`)

| Category | Technologies |
|---|---|
| **Framework** | Next.js 16 (App Router), React 19, TypeScript |
| **Styling** | Tailwind CSS v4, tw-animate-css, shadcn/ui (New York style) |
| **UI Libraries** | Radix UI primitives, Lucide & Tabler icons, motion (Framer Motion), GSAP, Lottie, Cobe |
| **Data Visualization** | D3.js, Recharts, react-syntax-highlighter, react-markdown |
| **State Management** | Zustand with persist middleware |
| **API & Integration** | MongoDB native driver, Pinecone SDK, OpenAI SDK, Nodemailer |
| **Search** | Custom cross-collection command palette (cmdk) |
| **Auth** | JWT (access + refresh tokens), bcryptjs, 3-layer auth (Edge, API, Client) |
| **Deployment** | Standalone Next.js output, cPanel Phusion Passenger, Vercel, Namecheap CI/CD |

---

## Database Schema

The system uses **MongoDB** as its primary database, shared between both packages. The server uses Beanie ODM for document modeling, while the admin panel uses the native MongoDB driver.

### 20+ Collections Overview

| Collection | Module | Purpose |
|---|---|---|
| `accounts` | Auth | User accounts with email, bcrypt-hashed password, role (`admin`/`user`) |
| `profiles` | Auth | Extended user profiles (name, student ID, department, batch, program, CGPA, credits) |
| `refresh_tokens` | Auth | JWT refresh token storage with revocation tracking |
| `agents` | Core Data / Agent Mgmt | AI agent configurations (name, prompt, active status) |
| `agent_tools` | Core Data | Per-agent tool bindings |
| `agent_configurations` | Core Data | Extended agent configuration JSON |
| `few_shot_examples` | Core Data | LLM query classification examples for RAG pipeline |
| `question_parts` | Core Data / Questions | Exam question metadata, text, image URLs, vector IDs for Pinecone |
| `user_threads` | Agentic Chat | Chat conversation threads with metadata |
| `chat_messages` | Agentic Chat | Individual chat messages within threads |
| `assessments` | Performance Tracker | Academic assessment records with marks and grades |
| `topics` | Performance Tracker | Course topic definitions with student confidence tracking |
| `student_courses` | Performance Tracker | Enrollment mappings with marks and computed grades |
| `courses` | Performance Tracker | Course catalog (code, title, credits, department) |
| `weaknesses` | Performance Tracker | Student weakness analysis records |
| `quiz_sessions` | Performance Tracker | Quiz attempts, submissions, and results |
| `performance_records` | Performance Tracker | Overall performance snapshots per student |
| `trimesters` | Performance Tracker | Trimester GPA records |
| `gpa_records` | Performance Tracker | Full GPA breakdowns with per-course grades |
| `student_events` | Student Events | Calendar events with date ranges |
| `roadmaps` | Roadmap Generator | Generated AI learning roadmaps with stages and items |
| `roadmap_node_progress` | Roadmap Generator | Per-node completion tracking |
| `roadmap_chat_messages` | Roadmap Generator | Roadmap-specific chat history |
| `system_logs` | Admin Panel | Immutable admin action audit trail |
| `notices` | Admin/User Portal | Published notices and announcements |

---

## API Endpoints

### Server API (FastAPI — 50+ endpoints)

**Auth** (`/auth`): Register, login, refresh tokens, logout, profile (GET/PATCH), forgot/reset password

**Agentic Chat** (`/chat`): Create/list/get/delete threads, send messages, simple chat, list available agents

**Grade Prediction** (`/api/v1/prediction`): Predict student grades via ensemble ML, health check

**Performance Tracker** (`/api/v1/performance`): Full CRUD for assessments, topics, courses, enrollments, weaknesses, quizzes, GPA calculation, CT count management

**Student Events** (`/api/v1/events`): Create, list (by student + date range), update, delete events

**Learning Roadmap** (`/roadmap`): Generate roadmaps, list/get/delete, chat with context, progress toggling

**Agent Management** (`/agents`): Admin CRUD for AI agent configurations

### Admin Panel API (Next.js — 30+ endpoints)

**Auth** (`/api/auth`): Login, register, logout, refresh, password reset with email

**Users** (`/api/users`): Paginated list with search/filter, create with auto-generated passwords, get/update/delete by ID, stats

**Courses** (`/api/courses`): CRUD with department tracking

**Course Management** (`/api/course-management`): Topics and trimesters CRUD

**Questions** (`/api/questions`): Paginated list with course/exam-type/semester filters, CRUD by ID, upload with Pinecone embedding generation

**Agents** (`/api/agents`, `/api/agent-tools`, `/api/agent-configurations`, `/api/few-shot-examples`): Full CRUD for agents and their configurations

**Search** (`/api/search`): Cross-collection search (users, questions, courses, agents, system-logs, navigation)

**Dashboard** (`/api/dashboard`): Analytics overview, question stats by course

**System Logs** (`/api/system-logs`): Filterable, paginated audit trail with admin/date/resource filters

**Embeddings** (`/api/update-embeddings`): Regenerate Pinecone embeddings for all questions or by course

**CDN** (`/api/cdn`): File upload and management for content delivery

**Notices & Academic Calendar** (`/api/notices`, `/api/academic-calendar`): CRUD for published content

---

## Security Architecture

The system implements a **multi-layered security model**:

| Layer | Mechanism |
|---|---|
| **Edge Middleware** | Next.js Edge Runtime validates JWT format, checks expiry, enforces role-based route access; redirects unauthenticated users to `/login` |
| **API Middleware** | Server-side JWT signature verification with `jsonwebtoken`, admin role enforcement, user object injection into request context |
| **Client Protection** | `ProtectedRoute` component with Zustand auth state check; redirects on authentication failure |
| **Token Strategy** | Short-lived access tokens (15 min) + long-lived refresh tokens (7 days) with rotation and revocation |
| **Password Security** | bcrypt hashing (12 rounds, 4096 iterations), secure auto-generated passwords (12 chars, 74-bit entropy), RFC email validation + disposable domain blocking |
| **Password Reset** | 3-minute time-limited reset tokens, separate signing secret, issuer/audience validation, single-use enforcement |
| **Cookie Security** | HttpOnly, Secure (production), SameSite=Lax configuration |
| **Audit Trail** | Immutable `system_logs` collection capturing before/after state, admin identity, IP, user-agent, and response timing for all POST/PUT/DELETE operations |
| **Error Handling** | Detailed server-side error logging with sanitized client responses — stack traces, file paths, and DB errors never exposed to users |

---

## AI & Machine Learning Highlights

### Language Model Integration
- **Primary Model**: GPT-4-turbo-preview (default) / GPT-3.5-turbo (fast fallback)
- **Embeddings**: OpenAI text-embedding-3-small / text-embedding-ada-002
- **Framework**: LangChain with LangGraph for stateful multi-agent orchestration
- **Tracking**: LangChain callback handler for LLM token usage monitoring

### RAG Pipeline Features
- **Few-Shot Learning**: Query classification examples stored in MongoDB, dynamically loaded per agent
- **Contextual Query Rewriting**: Conversation history injected into search queries for improved retrieval
- **Metadata Filtering**: Pinecone queries filtered by course code, exam type, and semester
- **Paginated Retrieval**: Stateful pagination with 50-question cap, dynamic quantity extraction from user queries
- **Graceful Degradation**: Falls back to conversation-only mode when LLM is unavailable

### ML Prediction Pipeline
```
Student Marks → Trimester Code Resolution → Enrollment History Fetch →
Feature Engineering (Course Clusters, CT Trends) →
Ensemble Prediction (RF + GB + MLP) →
Meta-Learning Adjustment (Fresh vs Experienced) →
Marking Scheme Application (Theory/Lab) → Grade Output
```

### Auto-Discovery Architecture
The server agent system uses dynamic filesystem scanning. Adding a new AI capability requires:
1. Create a Python file in `src/features/*/agent/` with a class extending `BaseAgent`
2. Insert a record in the MongoDB `agents` collection
3. The `AgentRegistry` discovers it automatically — no code changes in `main.py`

---

## UI & Experience Highlights

- **Animated Marketing Landing**: 3D globe (Cobe), GSAP scroll animations, Lottie illustrations, particle effects
- **Command Palette**: Cross-collection `cmd+k` search bar available throughout the admin panel
- **Dark Mode**: Full theme support via `next-themes` with system preference detection
- **Responsive Design**: Adaptive layouts for desktop and tablet, mobile chat sidebar with sheet drawer
- **Interactive Visualizations**: D3 force-directed roadmap graphs, Recharts analytics, radar charts for student performance
- **Streaming Chat**: Real-time AI response streaming with markdown rendering, syntax-highlighted code blocks, and copy functionality
- **Shimmer Animations**: Embedding processing progress with per-question shimmer effects
- **Toast Notifications**: Sonner-based toast system for action feedback
- **Frosted Glass UI**: Custom frosted header components with backdrop blur

---

## Deployment

The system is designed for modern cloud deployment:

| Component | Platform | Configuration |
|---|---|---|
| **FastAPI Server** | Vercel Serverless | Python 3.11, 250MB lambda, `vercel.json` routing |
| **Next.js Admin Panel** | Vercel / cPanel (Phusion Passenger) | `output: 'standalone'`, Node 20 |
| **Database** | MongoDB Atlas | Shared across both packages |
| **Vector Store** | Pinecone | Serverless index with metadata filtering |
| **CI/CD** | GitHub Actions | Namecheap deployment via rsync + PM2 |

The `.github/workflows/deploy.yml` pipeline automates build and deployment with environment-specific secrets for database URIs, JWT secrets, API keys, and email credentials.

---

## Project Structure (Detailed)

```
cosmos-its/
│
├── cosmos-its-server/
│   ├── api/
│   │   └── index.py                  # Vercel serverless entry
│   ├── src/
│   │   ├── main.py                   # FastAPI bootstrap, CORS, lifespan
│   │   ├── config/
│   │   │   ├── settings.py           # pydantic-settings (env vars)
│   │   │   └── constants.py          # LLM, retrieval, agent config
│   │   ├── core/
│   │   │   ├── infra/
│   │   │   │   ├── database.py       # Motor + Beanie ODM
│   │   │   │   ├── llm_client.py     # OpenAI/LangChain integration
│   │   │   │   ├── pinecone_client.py# Vector DB service
│   │   │   │   └── services.py       # DI container
│   │   │   ├── agent/
│   │   │   │   ├── registry.py       # Dynamic agent discovery
│   │   │   │   ├── factory.py        # Agent instantiation from DB
│   │   │   │   ├── base_agent.py     # Abstract agent class
│   │   │   │   ├── orchestrator.py   # LangGraph StateGraph
│   │   │   │   └── agent_state.py    # TypedDict state defs
│   │   │   ├── security/
│   │   │   │   └── jwt_utils.py      # JWT + bcrypt utilities
│   │   │   ├── data/
│   │   │   │   └── models.py         # Core MongoDB models
│   │   │   └── utils/
│   │   │       ├── logger.py         # Colored console logging
│   │   │       ├── exceptions.py     # Custom exception hierarchy
│   │   │       ├── title_generator.py
│   │   │       ├── llm_tracker.py    # Token usage tracking
│   │   │       ├── message_formatter.py
│   │   │       └── performance_context.py
│   │   └── features/
│   │       ├── auth/                 # Registration, login, profile
│   │       ├── agentic_chat/         # Supervisor, course, general agents
│   │       ├── prediction_model/     # Ensemble ML grade prediction
│   │       ├── performance_tracker/  # Assessments, quizzes, GPA
│   │       ├── student_events/       # Calendar management
│   │       ├── roadmap_generator/    # AI learning roadmaps
│   │       └── agent_management/     # Admin agent CRUD
│   ├── models/                       # Joblib ML artifacts
│   └── docs/                         # Run scripts, issue tracking
│
├── cosmos-admin-panel/
│   ├── app/
│   │   ├── (landing)/                # Marketing page
│   │   ├── login/                    # Auth pages
│   │   ├── register/
│   │   ├── reset-password/
│   │   ├── admin/                    # Admin dashboard & CRUD modules
│   │   ├── user/                     # Student portal (13+ pages)
│   │   ├── chat/                     # AI chat interface
│   │   └── api/                      # All server API routes
│   ├── components/
│   │   ├── ui/                       # shadcn primitives (button, card, etc.)
│   │   ├── dashboard/                # Stats cards, charts
│   │   ├── landing/                  # Marketing header/footer
│   │   ├── roadmap/                  # D3 graph components
│   │   ├── agents/                   # Agent form components
│   │   ├── kokonutui/               # External registry components
│   │   └── *.tsx                    # ProtectedRoute, ThemeProvider, GlobalSearch, etc.
│   ├── lib/                          # 28 service modules (auth, DB, AI, email, etc.)
│   ├── store/auth.ts                 # Zustand persistence store
│   ├── hooks/                        # use-mobile, use-auto-resize-textarea
│   └── middleware.ts                 # Edge auth + RBAC
│
└── README.md
```
