# JB Academic Publication Portal 

> **⚠️ Note:** This project is under active development. Some requirements, features, and implementation details may evolve as development progresses. Refer to the latest version of this document for the most current information.

[![Status](https://img.shields.io/badge/Status-Under%20Development-yellow)]()
[![Frontend](https://img.shields.io/badge/Frontend-Next.js-black)](https://nextjs.org/)
[![Backend](https://img.shields.io/badge/Backend-Spring%20Boot-green)](https://spring.io/projects/spring-boot)
[![AI Service](https://img.shields.io/badge/AI%20Service-FastAPI-009688)](https://fastapi.tiangolo.com/)
[![Database](https://img.shields.io/badge/Database-PostgreSQL-blue)](https://www.postgresql.org/)
[![Storage](https://img.shields.io/badge/Storage-MinIO-red)](https://min.io/)

---

## Table of Contents

- [Overview](#overview)
- [Technology Stack](#technology-stack)
- [System Roles](#system-roles)
- [Core System Modules](#core-system-modules)
  - [1. User Management and Authentication](#1-user-management-and-authentication)
  - [2. Submission Management](#2-submission-management)
  - [3. Review Workflow](#3-review-workflow)
  - [4. Publication and Repository](#4-publication-and-repository)
  - [5. Search and Discovery](#5-search-and-discovery)
  - [6. AI-Based Document Processing](#6-ai-based-document-processing)
- [System Architecture](#system-architecture)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Deployment Architecture](#deployment-architecture)
- [Development Approach](#development-approach)

---

## Overview

The **Academic Research and Project Submission Portal** is a web-based platform designed to manage the submission, review, and publication of academic work within a college environment.

The system supports multiple categories of academic output including:

- Research Papers
- Major Project Reports
- Mini Project Reports
- Internship Reports

The platform provides a structured workflow where:

1. **Students** submit academic work.
2. **Faculty members** review the submissions.
3. **Administrators or Heads of Department** make final decisions.
4. Approved submissions are **published** in a public academic repository.

The system also includes a dedicated **AI document processing service** responsible for PDF validation, text extraction, and semantic indexing to improve document discovery and analysis.

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js with TypeScript |
| Backend | Spring Boot (Java + Kotlin) |
| AI Service | Python FastAPI with Retrieval-Augmented Generation (RAG) |
| Database | PostgreSQL with pgvector extension |
| File Storage | MinIO (S3-compatible object storage) |

---

## System Roles

The platform defines four primary roles.

| Role | Description |
|------|-------------|
| **Guest** | Browse and access published academic content without authentication |
| **Student** | Submit academic work, track progress, view feedback, and submit revisions |
| **Faculty Reviewer** | Evaluate assigned submissions, provide feedback, and recommend decisions |
| **Admin / HOD** | Manage users, assign reviewers, make final decisions, and publish approved work |

### Use Case Diagram

```mermaid
graph TB
    subgraph "Guest Actions"
        Browse["Browse Publications"]
        Search["Search Publications"]
        View["View Paper Details"]
        Download["Download Documents"]
    end

    subgraph "Student Actions"
        Submit["Submit Academic Work"]
        Upload["Upload Documents"]
        Track["Track Submission Status"]
        Feedback["View Reviewer Feedback"]
        Revise["Submit Revisions"]
    end

    subgraph "Reviewer Actions"
        Review["Evaluate Submissions"]
        Score["Score Submission"]
        Recommend["Recommend Decision"]
    end

    subgraph "Admin Actions"
        Users["Manage Users"]
        Assign["Assign Reviewers"]
        Decision["Make Final Decision"]
        Publish["Publish Approved Work"]
        BulkImport["Bulk Import Users"]
    end

    Guest["Guest"] --> Browse
    Guest --> Search
    Guest --> View
    Guest --> Download

    Student["Student"] --> Submit
    Student --> Upload
    Student --> Track
    Student --> Feedback
    Student --> Revise

    Reviewer["Faculty Reviewer"] --> Review
    Reviewer --> Score
    Reviewer --> Recommend

    Admin["Admin / HOD"] --> Users
    Admin --> Assign
    Admin --> Decision
    Admin --> Publish
    Admin --> BulkImport
```

---

## Core System Modules

---

### 1. User Management and Authentication

The platform provides a centralized authentication and user management system.

**Key capabilities include:**

- Institutional account creation using roll number and institutional email
- Secure authentication using Spring Security and JWT
- Role-based access control
- Password change requirement during first login
- Profile management for students and faculty
- Administrative user management
- Bulk import of student accounts using CSV

#### Class Diagram — User Management

```mermaid
classDiagram
    class User {
        +Long id
        +String email
        +String passwordHash
        +String rollNumber
        +Role role
        +Boolean firstLogin
        +Boolean isActive
        +DateTime createdAt
        +DateTime updatedAt
    }

    class StudentProfile {
        +Long id
        +String fullName
        +String department
        +String batch
        +String academicYear
        +String phone
    }

    class FacultyProfile {
        +Long id
        +String fullName
        +String department
        +String designation
        +String specialization
    }

    class Role {
        <<enumeration>>
        GUEST
        STUDENT
        FACULTY_REVIEWER
        ADMIN
    }

    User "1" --> "1" Role
    User "1" --> "0..1" StudentProfile
    User "1" --> "0..1" FacultyProfile
```

#### Sequence Diagram — Authentication Flow

```mermaid
sequenceDiagram
    actor Student
    participant Frontend as Next.js Frontend
    participant Backend as Spring Boot Backend
    participant DB as PostgreSQL
    participant JWT as JWT Service

    Student->>Frontend: Enter credentials
    Frontend->>Backend: POST /api/auth/login
    Backend->>DB: Validate credentials
    DB-->>Backend: User record

    alt Authentication Success
        Backend->>DB: Check firstLogin flag

        alt First Login
            Backend-->>Frontend: Require password change
            Student->>Frontend: Enter new password
            Frontend->>Backend: POST /api/auth/change-password
            Backend->>DB: Update password
            Backend->>JWT: Generate tokens
            JWT-->>Backend: Access + Refresh tokens
            Backend-->>Frontend: 200 OK + tokens
        else Regular Login
            Backend->>JWT: Generate tokens
            JWT-->>Backend: Access + Refresh tokens
            Backend-->>Frontend: 200 OK + tokens
        end

        Frontend-->>Student: Redirect to dashboard
    else Authentication Failure
        Backend-->>Frontend: 401 Unauthorized
        Frontend-->>Student: Show error message
    end
```

---

### 2. Submission Management

The submission module allows students to upload and manage academic work.

**Supported categories:**

| Category | Description |
|----------|-------------|
| Research Paper | Original research contributions |
| Major Project Report | Final year or capstone project reports |
| Mini Project Report | Smaller scope academic projects |
| Internship Report | Industry internship documentation |

**Each submission includes:**

- Title, abstract, and keywords
- Author information
- Department and academic year
- Uploaded PDF document
- Optional supplementary files (source code, GitHub links, images, videos)

**Additional features:**

- Draft saving before final submission
- Version tracking for revised submissions
- Unique submission identifiers for tracking

#### Submission Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Draft : Student creates submission

    Draft --> Draft : Save draft
    Draft --> Submitted : Student submits

    Submitted --> UnderReview : Admin assigns reviewers

    UnderReview --> RevisionRequested : Reviewers recommend revisions
    UnderReview --> Accepted : Admin accepts
    UnderReview --> Rejected : Admin rejects

    RevisionRequested --> Revised : Student submits revision
    Revised --> UnderReview : Admin re-assigns for review

    Accepted --> Published : Admin publishes

    Published --> [*]
    Rejected --> [*]
```

#### Class Diagram — Submission

```mermaid
classDiagram
    class Submission {
        +Long id
        +String submissionId
        +String title
        +String abstractText
        +SubmissionCategory category
        +SubmissionStatus status
        +String department
        +String academicYear
        +DateTime createdAt
        +DateTime updatedAt
    }

    class SubmissionCategory {
        <<enumeration>>
        RESEARCH_PAPER
        MAJOR_PROJECT_REPORT
        MINI_PROJECT_REPORT
        INTERNSHIP_REPORT
    }

    class SubmissionStatus {
        <<enumeration>>
        DRAFT
        SUBMITTED
        UNDER_REVIEW
        REVISION_REQUESTED
        REVISED
        ACCEPTED
        REJECTED
        PUBLISHED
    }

    class Document {
        +Long id
        +String fileName
        +String fileType
        +Long fileSize
        +String minioObjectKey
        +Integer versionNumber
        +DateTime uploadedAt
    }

    class SupplementaryFile {
        +Long id
        +String fileName
        +String fileType
        +String url
        +String description
    }

    Submission "1" --> "1..*" Document : contains
    Submission "1" --> "0..*" SupplementaryFile : includes
    Submission "1" --> "1" SubmissionCategory
    Submission "1" --> "1" SubmissionStatus
```

---

### 3. Review Workflow

The system implements a structured academic review workflow.

**Workflow steps:**

1. A student submits academic work.
2. The administrator assigns one or more faculty reviewers.
3. Reviewers evaluate the submission and provide feedback.
4. Reviewers recommend a decision.
5. The administrator makes the final decision.

**Possible decisions:**

| Decision | Description |
|----------|-------------|
| **Accept** | Submission is approved for publication |
| **Revision Required** | Student must address feedback and resubmit |
| **Reject** | Submission is not accepted |

#### Review Workflow Diagram

```mermaid
sequenceDiagram
    actor Student
    actor Admin as Admin / HOD
    actor Reviewer as Faculty Reviewer
    participant Backend as Spring Boot Backend
    participant DB as PostgreSQL

    Student->>Backend: Submit academic work
    Backend->>DB: Save submission (status: SUBMITTED)
    Backend-->>Admin: Notify new submission

    Admin->>Backend: Assign reviewer(s)
    Backend->>DB: Create review assignment(s)
    Backend->>DB: Update status to UNDER_REVIEW
    Backend-->>Reviewer: Notify assignment

    Reviewer->>Backend: Fetch submission details
    Backend-->>Reviewer: Return submission and documents

    Reviewer->>Backend: Submit review (feedback + score + decision)
    Backend->>DB: Save review
    Backend-->>Admin: Notify review completed

    alt Decision: Accept
        Admin->>Backend: Accept submission
        Backend->>DB: Update status to ACCEPTED
        Backend-->>Student: Notify acceptance
    else Decision: Revision Required
        Admin->>Backend: Request revisions
        Backend->>DB: Update status to REVISION_REQUESTED
        Backend-->>Student: Notify revision needed
        Student->>Backend: Submit revised version
        Backend->>DB: Save new version (status: REVISED)
        Note over Admin, Reviewer: Re-review cycle begins
    else Decision: Reject
        Admin->>Backend: Reject submission
        Backend->>DB: Update status to REJECTED
        Backend-->>Student: Notify rejection
    end
```

#### Class Diagram — Review System

```mermaid
classDiagram
    class ReviewAssignment {
        +Long id
        +Long submissionId
        +Long reviewerId
        +String status
        +DateTime assignedAt
        +DateTime completedAt
    }

    class Review {
        +Long id
        +Long assignmentId
        +String feedbackComments
        +Integer overallScore
        +String recommendedDecision
        +DateTime submittedAt
    }

    class ReviewDecision {
        <<enumeration>>
        ACCEPT
        REVISION_REQUIRED
        REJECT
    }

    class EditorialDecision {
        +Long id
        +Long submissionId
        +Long adminId
        +String finalDecision
        +String comments
        +DateTime decidedAt
    }

    ReviewAssignment "1" --> "0..1" Review : produces
    Review --> ReviewDecision : recommends
    EditorialDecision --> ReviewDecision : decides
    Submission "1" --> "1..*" ReviewAssignment : assigned for
    Submission "1" --> "0..1" EditorialDecision : decided by
```

---

### 4. Publication and Repository

Submissions that are accepted are published in the institutional repository.

**Published records include:**

- Title and author information
- Abstract and keywords
- Downloadable PDF document
- Publication date
- View and download counts

**Content can be browsed by:**

- Department
- Academic year
- Submission category
- Author name

#### Class Diagram — Publication

```mermaid
classDiagram
    class Publication {
        +Long id
        +Long submissionId
        +String title
        +String abstractText
        +String department
        +String academicYear
        +String category
        +DateTime publishedAt
    }

    class PublicationMetrics {
        +Long id
        +Long publicationId
        +Long viewCount
        +Long downloadCount
        +DateTime lastAccessedAt
    }

    Publication "1" --> "1" PublicationMetrics : tracked by
    Publication "1" --> "1" Submission : derived from
```

---

### 5. Search and Discovery

The portal includes a search system for discovering academic content.

**Capabilities include:**

- Full-text search across indexed documents
- Filtering by department, year, author, or category
- Keyword-based discovery
- Ranked search results by relevance

The search system uses **PostgreSQL full-text search** with semantic enhancement from the AI service.

#### Search Flow Diagram

```mermaid
flowchart TD
    A([User initiates search]) --> B{Search Type?}

    B -->|Full-Text| C[Enter search query]
    B -->|Filtered Browse| D[Select filters]
    B -->|Keyword| E[Click keyword tag]

    C --> F[Send query to backend]
    D --> F
    E --> F

    F --> G{Semantic Search Enabled?}

    G -->|Yes| H[FastAPI: Generate query embedding]
    G -->|No| I[PostgreSQL: Full-text search]

    H --> J[pgvector: Cosine similarity search]
    J --> K[Combine and rank results]
    I --> K

    K --> L[Return paginated results]

    L --> M{User action?}
    M -->|View Details| N[Display publication]
    M -->|Download| O[Fetch from MinIO]
    M -->|Refine| B
    M -->|Done| P([End])
```

---

### 6. AI-Based Document Processing

Document analysis and semantic processing are handled by a dedicated **Python service built with FastAPI**.  
This service is responsible for validating uploaded documents, extracting meaningful information, and enabling AI-assisted discovery of academic content.

The AI service operates as an auxiliary component of the backend and is invoked whenever a new document is uploaded or when semantic search is requested.

**Key capabilities include:**

- Validation of uploaded PDF documents (format, integrity, and basic compliance checks)
- Extraction of textual content and structural metadata from documents
- Generation of semantic vector embeddings for document content
- Automatic generation of concise document summaries
- Support for semantic search and related document recommendations

The generated embeddings are stored in **PostgreSQL using the `pgvector` extension**, allowing efficient vector similarity searches across the document corpus.

---

#### AI Processing Pipeline

```mermaid
sequenceDiagram
    participant Backend as Spring Boot Backend
    participant Storage as MinIO Object Storage
    participant AI as FastAPI AI Service
    participant DB as PostgreSQL (pgvector)

    Backend->>Storage: Upload PDF document
    Storage-->>Backend: Return object key

    Backend->>AI: POST /process-document (objectKey)

    AI->>Storage: Retrieve PDF document
    Storage-->>AI: PDF file

    Note over AI: Document Processing Pipeline

    AI->>AI: Validate PDF format and structure
    AI->>AI: Extract text content
    AI->>AI: Extract structural metadata
    AI->>AI: Generate semantic embeddings
    AI->>AI: Generate document summary

    AI->>DB: Store extracted metadata
    AI->>DB: Store vector embeddings (pgvector)
    AI->>DB: Store generated summary

    AI-->>Backend: Processing completed

    Note over Backend,DB: Semantic Search Workflow

    Backend->>AI: POST /search (query text)
    AI->>AI: Generate query embedding
    AI->>DB: Perform vector similarity search
    DB-->>AI: Return ranked matches
    AI-->>Backend: Return search results
```

#### Class Diagram — AI Service

```mermaid
classDiagram
    class DocumentProcessor {
        +processDocument(objectKey, submissionId)
        -validatePDF(binary)
        -extractText(binary)
        -extractStructure(text)
    }

    class PDFValidator {
        +validate(binary)
        +checkFormat()
        +checkIntegrity()
    }

    class TextExtractor {
        +extract(binary)
        +extractByPage(binary, pageNum)
    }

    class EmbeddingGenerator {
        +generateEmbedding(text)
        +generateChunkEmbeddings(chunks)
        +chunkText(text, chunkSize, overlap)
    }

    class SummaryGenerator {
        +generateSummary(text)
        +generateAbstract(text)
    }

    class SemanticSearchService {
        +search(queryText, filters)
        +findRelatedDocuments(submissionId, topK)
    }

    class DocumentEmbedding {
        +Long id
        +Long submissionId
        +String chunkText
        +int chunkIndex
        +vector embedding
        +DateTime createdAt
    }

    DocumentProcessor --> PDFValidator
    DocumentProcessor --> TextExtractor
    DocumentProcessor --> EmbeddingGenerator
    DocumentProcessor --> SummaryGenerator
    EmbeddingGenerator ..> DocumentEmbedding : produces
    SemanticSearchService --> EmbeddingGenerator
```

---

## System Architecture

The system follows a modular architecture consisting of a web frontend, a backend application, and a specialized document processing service.

```mermaid
graph TB
    Browser["Client Browser"]

    subgraph "Presentation Layer"
        Frontend["Next.js Frontend (TypeScript)"]
    end

    subgraph "Application Layer"
        Backend["Spring Boot Backend (Java + Kotlin)"]
    end

    subgraph "AI Processing Layer"
        AI["FastAPI AI Service (Python)"]
    end

    subgraph "Data Layer"
        DB[("PostgreSQL + pgvector")]
        Storage[("MinIO Object Storage")]
    end

    Browser <-->|HTTPS| Frontend
    Frontend <-->|REST API| Backend

    Backend --> DB
    Backend --> Storage
    Backend <-->|REST API| AI

    AI --> DB
    AI --> Storage
```

**Architecture Flow:**

```
Client Browser
      │
      ▼
Next.js Frontend (TypeScript)
      │
      ▼
Spring Boot Backend (Java + Kotlin)
      │
      ├── PostgreSQL (application data + pgvector)
      ├── MinIO (document storage)
      └── Python FastAPI Service (document processing + AI)
```

---

## Entity Relationship Diagram

```mermaid
erDiagram
    USER {
        bigint id PK
        varchar email UK
        varchar password_hash
        varchar roll_number UK
        varchar role
        boolean first_login
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    STUDENT_PROFILE {
        bigint id PK
        bigint user_id FK
        varchar full_name
        varchar department
        varchar batch
        varchar academic_year
        varchar phone
    }

    FACULTY_PROFILE {
        bigint id PK
        bigint user_id FK
        varchar full_name
        varchar department
        varchar designation
        varchar specialization
    }

    SUBMISSION {
        bigint id PK
        varchar submission_id UK
        varchar title
        text abstract_text
        varchar category
        varchar status
        bigint author_id FK
        varchar department
        varchar academic_year
        timestamp created_at
        timestamp updated_at
    }

    DOCUMENT {
        bigint id PK
        bigint submission_id FK
        varchar file_name
        varchar file_type
        bigint file_size
        varchar minio_object_key
        int version_number
        timestamp uploaded_at
    }

    SUPPLEMENTARY_FILE {
        bigint id PK
        bigint submission_id FK
        varchar file_name
        varchar file_type
        varchar url
        text description
    }

    REVIEW_ASSIGNMENT {
        bigint id PK
        bigint submission_id FK
        bigint reviewer_id FK
        varchar status
        timestamp assigned_at
        timestamp completed_at
    }

    REVIEW {
        bigint id PK
        bigint assignment_id FK
        text feedback_comments
        int overall_score
        varchar recommended_decision
        timestamp submitted_at
    }

    EDITORIAL_DECISION {
        bigint id PK
        bigint submission_id FK
        bigint admin_id FK
        varchar final_decision
        text comments
        timestamp decided_at
    }

    PUBLICATION {
        bigint id PK
        bigint submission_id FK
        varchar title
        text abstract_text
        varchar department
        varchar academic_year
        varchar category
        timestamp published_at
    }

    PUBLICATION_METRICS {
        bigint id PK
        bigint publication_id FK
        bigint view_count
        bigint download_count
        timestamp last_accessed_at
    }

    DOCUMENT_EMBEDDING {
        bigint id PK
        bigint submission_id FK
        text chunk_text
        int chunk_index
        vector embedding
        timestamp created_at
    }

    USER ||--o| STUDENT_PROFILE : has
    USER ||--o| FACULTY_PROFILE : has
    USER ||--o{ SUBMISSION : authors
    SUBMISSION ||--o{ DOCUMENT : contains
    SUBMISSION ||--o{ SUPPLEMENTARY_FILE : includes
    SUBMISSION ||--o{ REVIEW_ASSIGNMENT : assigned_for
    REVIEW_ASSIGNMENT ||--o| REVIEW : produces
    REVIEW_ASSIGNMENT }o--|| USER : reviewer
    SUBMISSION ||--o| EDITORIAL_DECISION : decided_by
    EDITORIAL_DECISION }o--|| USER : admin
    SUBMISSION ||--o| PUBLICATION : published_as
    PUBLICATION ||--|| PUBLICATION_METRICS : tracked_by
    SUBMISSION ||--o{ DOCUMENT_EMBEDDING : indexed_with
```

---

## Deployment Architecture

```mermaid
graph TB
    Users["Users"] -->|HTTPS : 443| LB["Nginx / Reverse Proxy"]

    LB -->|Port 3000| NextJS["Next.js Frontend"]
    NextJS -->|Port 8080| SpringBoot["Spring Boot Backend"]

    SpringBoot -->|Port 5432| PostgreSQL[("PostgreSQL + pgvector")]
    SpringBoot -->|Port 9000| MinIO[("MinIO Object Storage")]
    SpringBoot -->|Port 8000| FastAPI["FastAPI AI Service"]

    FastAPI -->|Port 5432| PostgreSQL
    FastAPI -->|Port 9000| MinIO
```

**Port Reference:**

| Service | Port | Protocol |
|---------|------|----------|
| Nginx (Reverse Proxy) | 443 | HTTPS |
| Next.js Frontend | 3000 | HTTP |
| Spring Boot Backend | 8080 | HTTP |
| FastAPI AI Service | 8000 | HTTP |
| PostgreSQL | 5432 | TCP |
| MinIO API | 9000 | HTTP |
| MinIO Console | 9001 | HTTP |

---

## Development Approach

Development will proceed incrementally with an emphasis on delivering a stable core platform first.

**Initial implementation focuses on:**

- Authentication and role management
- Submission workflow
- Review system
- Publication repository
- Search functionality
- AI-based document processing

**Future enhancements may include:**

- Advanced analytics and reporting dashboards
- Plagiarism detection
- Citation analysis
- Integration with external academic systems (ORCID, CrossRef)
- Email notification system
- Mobile-responsive progressive web app

---
