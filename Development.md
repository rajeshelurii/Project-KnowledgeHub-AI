Product Discovery:

Project: KnowledgeHub AI

KnowledgeHub AI is an enterprise platform that allows organizations to securely upload internal documents and enables employees to retrieve accurate information through AI-powered semantic search and conversational chat.

Our application will:
Store company documents.
Understand their meaning using embeddings.
Search semantically.
Use an LLM to generate grounded answers.
Cite the source documents.

Our Engineering Principles

These will guide every technical decision:
1. Security First
Company documents may be confidential.
2. Performance Matters
Users should get answers quickly.
3. Scalability
The system should be designed so it could grow from hundreds to many thousands of documents without fundamental redesign.
4. Maintainability
Clean, modular code that is easy to extend.
5. Explainability
Every AI answer should show where the information came from.

High-Level System Design (HLD)

Browser
    │
    ▼
Angular Application
    │
    ▼
ASP.NET Core Web API

Instead of putting everything into one giant service, we'll split responsibilities.

                 Angular
                    │
                    ▼
          ASP.NET Core API
                    │
 ┌──────────┬───────────┬───────────┬─────────────┐
 │          │           │           │             │
 ▼          ▼           ▼           ▼             ▼
Auth     Document      Chat      Search      Admin
Service   Service     Service    Service    Service

Each service has one responsibility.
This is called the Single Responsibility Principle (SRP).


Where do data go
                 Angular
                    │
                    ▼
          ASP.NET Core API
                    │
     ┌──────────────┴──────────────┐
     ▼                             ▼
 PostgreSQL                 Blob Storage

PostgreSQL stores
Users
Roles
Chats
Metadata
Audit logs
Document information

Blob Storage stores
The actual files.
HRPolicy.pdf
EmployeeHandbook.pdf
SecurityGuide.pdf

Because databases are not ideal for storing large files. We store metadata in PostgreSQL and the file itself in blob storage.

let's introduce AI.
When a PDF is uploaded:
Upload PDF
      │
      ▼
Blob Storage
      │
      ▼
Background Worker

The API does not process the PDF immediately.
If the API waits to:

Read the file
Extract text
Chunk it
Generate embeddings

the user could be waiting a long time.

Instead:
Upload
↓

Saved

↓

Response:
"Upload successful"

Meanwhile:
Background Worker

↓

Processes document

↓

Creates embeddings

↓

Stores vectors

The user gets a fast response while heavy work happens asynchronously.

Where do the vectors go?
We'll introduce another component.
Background Worker
       │
       ▼
Embedding Generator
       │
       ▼
PostgreSQL + pgvector

This means PostgreSQL stores both:
Relational data (users, documents, chats)
Vector embeddings

What happens when someone asks a question?
  User
  
  ↓
  
  Angular
  
  ↓
  
  Chat API
  
  ↓
  
  Vector Search
  
  ↓
  
  Top 5 Relevant Chunks
  
  ↓
  
  OpenAI
  
  ↓
  
  Answer
  
  ↓
  
  Angular

This is the heart of the application.

RAG stands for Retrieval-Augmented Generation.

Session 3 — Low-Level Design (LLD)
Now we stop thinking about services and start thinking about objects.
Imagine we're about to create a new company account.
What information do we need?
Immediately, we can identify our first entity.

Entity 1: User
User
-----------------------
Id
FirstName
LastName
Email
PasswordHash
RoleId
CreatedAt
UpdatedAt
LastLoginAt
IsActive

Notice something important. We do not store: Password Instead we store PasswordHash
Because if someone gains access to the database, we don't want them to see everyone's passwords. We'll discuss hashing in detail when we build authentication.

Entity 2: Role
Role
----------------------
Id
Name
Description

Instead of storing user we store and categorize to identify
Admin
Manager
Employee

This is called normalization and avoids duplicate data.

Entity 3: Document
When someone uploads a PDF.
We don't store the PDF inside PostgreSQL.
We store its information.
Document
-----------------------
Id
FileName
BlobUrl
UploadedBy
Size
ContentType
Status
UploadedAt

the actual file lives in Azure Blob Storage which can be redirected using BlobUrl.

Why Status? when someone uploads it may take time so we store in this Status = Processing or other to reflect the state of the file
The fromtend can show
✅ Ready
⏳ Processing
❌ Failed

Entity 4: DocumentChunk
This is where AI begins.
Suppose our document has 100 pages.
We split it into chunks.
Each chunk becomes one record.
DocumentChunk
--------------------------
Id
DocumentId
ChunkNumber
Content
Embedding
Example:
Chunk 1 Employees receive 24 annual leaves...
Chunk 2 Employees receive 12 sick leaves... and so on

Why store chunks separately?
Imagine a document with 800 pages. When someone asks: "How many sick leaves?" We don't want to search all 800 pages. We search the chunks. Much faster.

Entity 5: Conversation
Conversation
-----------------------
Id
UserId
Title
CreatedAt

Example HR Questions
Project Documentation
API Help

Entity 6: Message
Every question and answer
Message
------------------------
Id
ConversationId
Role
Content
CreatedAt

Role can be User or Assistant or ...

Entity 7: Audit Log
This is something many portfolio projects ignore. Every important action gets recorded.
AuditLog
-------------------------
Id
UserId
Action
Entity
CreatedAt

Example Rajesh Uploaded Document HRPolicy.pdf 10:45 AM or Admin Deleted User John 2:10 PM

Enterprise applications almost always have some form of audit trail.

Relationships:
Let's connect everything.
  User
   │
   ├───────< Document
   │
   ├───────< Conversation
   │
   └───────< AuditLog
  
  Conversation
   │
   └────────< Message
  
  Document
   │
   └────────< DocumentChunk
  
  Role
   │
   └────────< User

This is a simple Entity Relationship Diagram (ERD).

Why are we designing this first?
Because the database reflects the business. Why are we designing this first? Because the database reflects the business.

Session 4 – Authentication & Security
Why do we need Authentication? magine our application has no login. Anyone can: Upload company documents Read HR policies Chat with AI Download confidential files Delete documents
So the first question our application asks is:
"Who are you?"
That's Authentication.

Authentication vs Authorization
Authentication Who are you?
Authorization what are you allowed to do based on role?

What happens when you click Login?
Angular sends: POST /api/auth/login with login creds
ASP.NET Core receives the request. it looks up User Email PasswordHash
Why Hash? magine a hacker steals the database. Game over. instead we use Hash Rajesh AQJ89SJKD8239...
Even we, as developers, cannot read it.

Login Successful
Should we ask for the password on every request? That would be a terrible user experience.
Enter JWT
JWT stands for JSON Web Token. Think of it as a digital ID card.
After login:
The server creates a token.
Example
UserId: 123
Email: rajesh@company.com
Role: Employee
Expires: 30 min

The actual token is encoded and signed, so the client can't tamper with it.

Angular stores this token securely.
Every API request includes it.
Authorization
Bearer eyJhbGciOi...

Why is JWT useful?
Instead of asking:
"Who are you?" every time, the API checks the token. Much faster.

But why does JWT expire? Imagine you lose your laptop. Someone opens it. If the token never expires... They stay logged in forever. Not good. So we make tokens short-lived.
Example: 30 minutes.

Then users would keep logging in?
Exactly.  hat's where Refresh Tokens come in.

Refresh Token
Think of it like this.

JWT

↓

Temporary visitor pass

Refresh Token

↓

Permanent badge stored securely

When JWT expires:
Angular silently asks:
I have a valid Refresh Token.
Can I get a new JWT?
Server says:
Yes.
Here's another 30-minute token.
The user never notices.

Why not make JWT valid for 30 days? Security. If someone steals it, they have 30 days of access. Short-lived access tokens reduce that risk.

Our Authentication Flow
  User
  
  ↓
  
  Login
  
  ↓
  
  Server verifies password
  
  ↓
  
  Generate JWT
  
  ↓
  
  Generate Refresh Token
  
  ↓
  
  Angular stores tokens
  
  ↓
  
  API Requests
  
  ↓
  
  JWT Verified
  
  ↓
  
  Access Granted


Where does Authorization happen?
Suppose Rajesh tries: DELETE /api/users/10
The API reads the JWT: Role Employee
Endpoint requires: Admin

Result: 403 Forbidden
The request is authenticated but not authorized.

One More Security Layer
Remember our AI? Suppose an Employee asks:
"Show me HR salary documents."
Before searching vectors, the application checks:

Is the user authenticated? What role do they have? Which documents can they access?
Only authorized documents are searched.

This means the AI never even receives restricted information.

Why is this architecture important?
By now, you can see the pattern. Every feature in our system follows a pipeline:

  Request
  
  ↓
  
  Authentication
  
  ↓
  
  Authorization
  
  ↓
  
  Business Logic
  
  ↓
  
  Database / Blob / AI
  
  ↓
  
  Response

Nothing bypasses security.

Session 5 — Clean Architecture

First, I want to ask you a question. Suppose we build our application like this:
Controllers

↓

Everything

Example:
DocumentController
- Upload PDF
- Save Database
- Generate Embeddings
- Call OpenAI
- Send Email
- Write Logs
- Validate User

Looks easy, right? But imagine six months later.
The controller becomes: 2,000 lines.
Now someone says: "Replace OpenAI with Azure OpenAI."
You start changing code inside the controller.
Then another developer says: "We need AWS S3 instead of Azure Blob."
More changes.
Eventually: Everything depends on everything.
This is called a Big Ball of Mud.
Many real projects become like this.

So what's the solution? Instead of organizing by files, we organize by responsibilities.

Clean Architecture We'll split our project into layers.
             ┌─────────────────────────┐
             │   PRESENTATION LAYER    │
             │  (Controllers, Routes)  │
             └────────────┬────────────┘
                          │ (Calls)
                          ▼
┌───────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                   │
│         (Use Cases, DTOs, Interface Blueprints)       │
└─────────────────────────┬─────────────────────────────┘
                          │ (Orchestrates)
                          ▼
┌───────────────────────────────────────────────────────┐
│                     DOMAIN LAYER                      │
│        (Entities, Value Objects, Core Rules)          │
└───────────────────────────────────────────────────────┘
                          ▲
                          │ (Implements Interfaces)
             ┌────────────┴────────────┐
             │  INFRASTRUCTURE LAYER   │
             │  (PostgreSQL, AWS S3)   │
             └─────────────────────────┘

The Flow
Suppose Rajesh uploads a PDF.
  Angular
  
  ↓
  
  Presentation
  
  ↓
  
  Application
  
  ↓
  
  Infrastructure
  
  ↓
  
  Blob Storage
  
  ↓
  
  Database

Notice something. The request always moves inward.
Why is this important?
Imagine tomorrow.
Your company says: "We don't want Azure Blob anymore."
Instead: AWS S3
Where do we change code?
Only here: Infrastructure
Everything else stays the same.
That is one of the biggest benefits of this architecture.

Another Example
Today: OpenAI
Tomorrow: Azure OpenAI
Or
Gemini
Or
Claude
Do we rewrite the Application layer? No.
Only the Infrastructure implementation changes.

Why is this used in Enterprise?
Because enterprise systems live for years.
Technologies change.
Business rules change much more slowly.

Dependency Rule
Outer layers depend on inner layers.
Inner layers never depend on outer layers.

What does this mean for our project?
We'll likely have a solution structure similar to this:
KnowledgeHubAI.sln
├── KnowledgeHubAI.Api
│
├── KnowledgeHubAI.Application
│
├── KnowledgeHubAI.Domain
│
├── KnowledgeHubAI.Infrastructure
│
└── KnowledgeHubAI.Tests
Each project has a clear responsibility.

Why this matters for your career
Many developers can build a CRUD app.
Fewer can explain why a layered architecture improves maintainability, testability, and flexibility.
When an interviewer asks:
"Why did you choose Clean Architecture?"
You won't answer:
"Because it's popular."
You'll answer:
"Because it keeps business logic independent of frameworks and external services, making the application easier to test, maintain, and evolve. For example, if we switch from Azure Blob Storage to Amazon S3, only the Infrastructure layer changes."




