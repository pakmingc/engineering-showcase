# Architecture Diagrams

This document contains anonymized architecture diagrams representing patterns I've implemented in production systems.

---

## 1. Full-Stack SaaS Architecture

<img src="images/saas-architecture.png.jpeg" alt="SaaS Architecture" width="700">

A typical architecture for AI-powered SaaS applications I build.

```mermaid
flowchart TB
    subgraph Client["Client Layer"]
        Browser["React/Vite SPA"]
        Mobile["Mobile Web (PWA)"]
    end

    subgraph Edge["Edge/CDN"]
        CDN["Vercel/Cloudflare"]
    end

    subgraph API["API Layer"]
        Gateway["API Gateway"]
        Auth["Auth Middleware"]
        RateLimit["Rate Limiter"]
    end

    subgraph Services["Service Layer"]
        CoreAPI["Core API (Flask/Node)"]
        AIService["AI Service"]
        ProcessingWorker["Background Worker"]
    end

    subgraph Data["Data Layer"]
        Postgres["PostgreSQL (Supabase)"]
        Redis["Redis Cache"]
        Storage["Object Storage (S3)"]
    end

    subgraph External["External Services"]
        LLM["LLM Provider"]
        Payment["Stripe"]
        Email["Email Service"]
    end

    Browser --> CDN
    Mobile --> CDN
    CDN --> Gateway
    Gateway --> Auth
    Auth --> RateLimit
    RateLimit --> CoreAPI
    CoreAPI --> AIService
    CoreAPI --> ProcessingWorker
    CoreAPI --> Postgres
    CoreAPI --> Redis
    CoreAPI --> Storage
    AIService --> LLM
    CoreAPI --> Payment
    CoreAPI --> Email
```

### Key Design Decisions

- **Edge-first delivery** for static assets and initial page loads
- **Middleware chain** for auth, rate limiting before hitting business logic
- **Background workers** for long-running AI tasks (avoid timeout issues)
- **Caching layer** for LLM responses and computed data

---

## 2. AI/LLM Integration Architecture

<img src="images/llm-integration-flow.png.jpeg" alt="LLM Integration Architecture" width="700">

Pattern for building reliable LLM-powered features with fallback and monitoring.

```mermaid
flowchart LR
    subgraph Input["Input Processing"]
        Request["User Request"]
        Validate["Input Validation"]
        Sanitize["Sanitization"]
    end

    subgraph Router["LLM Router"]
        Primary["Primary Provider"]
        Fallback1["Fallback Provider 1"]
        Fallback2["Fallback Provider 2"]
    end

    subgraph Output["Output Processing"]
        Parse["Response Parser"]
        RefusalCheck["Refusal Detection"]
        Format["Response Formatter"]
    end

    subgraph Monitor["Monitoring"]
        Latency["Latency Tracking"]
        Errors["Error Logging"]
        Cost["Cost Tracking"]
    end

    Request --> Validate
    Validate --> Sanitize
    Sanitize --> Primary
    Primary -->|"Error/Timeout"| Fallback1
    Fallback1 -->|"Error/Timeout"| Fallback2
    Primary --> Parse
    Fallback1 --> Parse
    Fallback2 --> Parse
    Parse --> RefusalCheck
    RefusalCheck -->|"Refusal Detected"| Fallback1
    RefusalCheck --> Format

    Primary -.-> Latency
    Primary -.-> Errors
    Primary -.-> Cost
```

### Key Features

1. **Multi-provider fallback** - Automatic failover when primary provider fails
2. **Refusal detection** - Detect when LLM refuses to answer and retry with different provider
3. **Cost tracking** - Monitor spend per provider for optimization
4. **Response validation** - Ensure output matches expected format

---

## 3. Document Processing Pipeline

<img src="images/document-processing.png.jpeg" alt="Document Processing Pipeline" width="700">

Architecture for OCR and document processing systems.

```mermaid
flowchart TB
    subgraph Input["Document Input"]
        Upload["File Upload"]
        URL["URL Fetch"]
        Camera["Camera Capture"]
    end

    subgraph Preprocessing["Preprocessing"]
        Validate["File Validation"]
        Convert["Format Conversion"]
        Enhance["Image Enhancement"]
    end

    subgraph OCR["OCR Processing"]
        Engine1["Engine A (Paddle)"]
        Engine2["Engine B (Tesseract)"]
        Confidence["Confidence Scoring"]
        Merge["Result Merger"]
    end

    subgraph PostProcess["Post-Processing"]
        Clean["Text Cleaning"]
        AIFormat["AI Formatting"]
        Structure["Structure Extraction"]
    end

    subgraph Output["Output"]
        JSON["JSON Response"]
        PDF["PDF Generation"]
        Export["Export (DOCX/TXT)"]
    end

    Upload --> Validate
    URL --> Validate
    Camera --> Validate
    Validate --> Convert
    Convert --> Enhance
    Enhance --> Engine1
    Enhance --> Engine2
    Engine1 --> Confidence
    Engine2 --> Confidence
    Confidence --> Merge
    Merge --> Clean
    Clean --> AIFormat
    AIFormat --> Structure
    Structure --> JSON
    Structure --> PDF
    Structure --> Export
```

### Multi-Engine Strategy

- **Run multiple OCR engines in parallel** for better accuracy
- **Confidence scoring** to select best result
- **Fallback logic** if primary engine fails
- **AI post-processing** to clean and structure raw OCR output

---

## 4. Background Worker Architecture

Pattern for handling long-running tasks without blocking user requests.

```mermaid
flowchart LR
    subgraph API["API Server"]
        Endpoint["Task Endpoint"]
        TaskID["Generate Task ID"]
        Enqueue["Queue Task"]
    end

    subgraph Queue["Task Queue"]
        Redis["Redis Queue"]
        DLQ["Dead Letter Queue"]
    end

    subgraph Workers["Worker Pool"]
        Worker1["Worker 1"]
        Worker2["Worker 2"]
        Worker3["Worker N"]
    end

    subgraph Storage["Result Storage"]
        DB["Database"]
        Files["File Storage"]
    end

    subgraph Client["Client Polling"]
        Poll["Status Endpoint"]
        Webhook["Webhook Callback"]
    end

    Endpoint --> TaskID
    TaskID --> Enqueue
    Enqueue --> Redis
    Redis --> Worker1
    Redis --> Worker2
    Redis --> Worker3
    Worker1 -->|"Failure"| DLQ
    Worker1 --> DB
    Worker1 --> Files
    DB --> Poll
    Files --> Poll
    DB --> Webhook
```

### Implementation Notes

- **Immediate task ID return** - User gets ID instantly, polls for result
- **Idempotent workers** - Safe to retry failed tasks
- **Dead letter queue** - Failed tasks don't block the system
- **Webhook option** - For clients that prefer push over poll

---

## 5. Authentication & Authorization Flow

Typical auth flow for SaaS applications with subscription tiers.

```mermaid
sequenceDiagram
    participant User
    participant App
    participant Auth as Auth Provider
    participant API
    participant Stripe
    participant DB

    User->>App: Click Login
    App->>Auth: Redirect to OAuth
    Auth->>User: Show Login Form
    User->>Auth: Enter Credentials
    Auth->>App: Return JWT Token
    App->>API: Request with JWT
    API->>Auth: Verify Token
    Auth->>API: Token Valid + User ID
    API->>DB: Get User Subscription
    DB->>API: Subscription Tier
    API->>API: Check Feature Access
    API->>App: Response (or 403)

    Note over User,Stripe: Subscription Flow
    User->>App: Click Subscribe
    App->>API: Create Checkout Session
    API->>Stripe: Create Session
    Stripe->>API: Session URL
    API->>App: Redirect URL
    App->>Stripe: Redirect User
    User->>Stripe: Complete Payment
    Stripe->>API: Webhook: payment_success
    API->>DB: Update Subscription
```

### Security Considerations

- **JWT tokens** with short expiry, refresh token pattern
- **Server-side subscription check** on every protected request
- **Webhook signature verification** for Stripe events
- **Feature flags** tied to subscription tier

---

## 6. Computer Vision / Image Analysis Pipeline

<img src="images/testing-architecture.png.jpeg" alt="Vision Testing Architecture" width="700">

Architecture for floor plan analysis and image processing systems.

```mermaid
flowchart TB
    subgraph Input["Image Input"]
        Upload["File Upload"]
        Camera["Camera Capture"]
        URL["URL Import"]
    end

    subgraph Preprocessing["Preprocessing"]
        Validate["Format Validation"]
        Resize["Image Resize/Normalize"]
        Enhance["Contrast/Brightness"]
    end

    subgraph Vision["Vision Processing"]
        CloudVision["Google Cloud Vision API"]
        OpenCV["OpenCV Processing"]
        OCR["Pytesseract OCR"]
    end

    subgraph Analysis["Analysis Layer"]
        WallDetect["Wall Detection"]
        RoomDetect["Room Segmentation"]
        ScaleCalib["Scale Calibration"]
        SkeletonGraph["Graph Structure"]
    end

    subgraph Output["Output Generation"]
        JSON["Structured JSON"]
        Report["PDF Report"]
        Overlay["Annotated Image"]
    end

    Upload --> Validate
    Camera --> Validate
    URL --> Validate
    Validate --> Resize
    Resize --> Enhance
    Enhance --> CloudVision
    Enhance --> OpenCV
    CloudVision --> WallDetect
    OpenCV --> WallDetect
    OpenCV --> RoomDetect
    OCR --> ScaleCalib
    WallDetect --> SkeletonGraph
    RoomDetect --> SkeletonGraph
    ScaleCalib --> SkeletonGraph
    SkeletonGraph --> JSON
    SkeletonGraph --> Report
    SkeletonGraph --> Overlay
```

### Key Components

- **Multi-source input** - Upload, camera, or URL fetch
- **Google Cloud Vision** for text detection and image labeling
- **OpenCV** for contour detection, morphological operations
- **Pytesseract** for OCR of measurements and labels
- **Skeleton graph** for structural representation
- **Multiple outputs** - JSON data, annotated images, PDF reports

---

## 7. Document Generation Pipeline

Architecture for converting content to various document formats.

```mermaid
flowchart LR
    subgraph Input["Content Input"]
        Markdown["Markdown/MDX"]
        HTML["HTML Content"]
        JSON["Structured Data"]
    end

    subgraph Process["Processing"]
        Parse["Parser"]
        Transform["Transform"]
        Render["Renderer"]
    end

    subgraph Engines["Format Engines"]
        PDFLIB["pdf-lib"]
        JSPDF["jsPDF"]
        Puppeteer["Puppeteer"]
        Mammoth["Mammoth"]
    end

    subgraph Output["Output Formats"]
        PDF["PDF Document"]
        DOCX["Word Document"]
        XLSX["Excel Spreadsheet"]
        EPUB["EPUB eBook"]
    end

    Markdown --> Parse
    HTML --> Parse
    JSON --> Parse
    Parse --> Transform
    Transform --> Render
    Render --> PDFLIB
    Render --> JSPDF
    Render --> Puppeteer
    Render --> Mammoth
    PDFLIB --> PDF
    JSPDF --> PDF
    Puppeteer --> PDF
    Mammoth --> DOCX
    Render --> XLSX
    Render --> EPUB
```

### Implementation Notes

- **pdf-lib** for programmatic PDF creation and manipulation
- **jsPDF** for client-side PDF generation
- **Puppeteer** for HTML-to-PDF with full CSS support
- **mammoth** for Word document parsing, **docx** for generation
- **xlsx** for Excel file handling
- **Mermaid** and **KaTeX** integration for diagrams and math

---

## Next Steps

These diagrams represent patterns, not specific implementations. In an interview, I can:

1. Dive deeper into any component
2. Discuss alternative approaches I considered
3. Explain specific challenges I faced
4. Show how these patterns adapt to different requirements
