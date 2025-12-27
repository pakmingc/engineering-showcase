# Technical Decision Log

This document captures significant technical decisions I've made across projects. Each entry follows a structured format to show my reasoning process.

---

## 1. Choosing Supabase Over Firebase for Project X

**Date:** 2024

### Context
Building a new SaaS application that needed user authentication, real-time features, and a relational database. Had to choose between Firebase (familiar) and Supabase (PostgreSQL-based).

### Options Considered
1. **Firebase** - Good ecosystem, real-time by default, NoSQL
2. **Supabase** - PostgreSQL, SQL queries, Row Level Security
3. **Custom Auth + Postgres** - Maximum control, more setup

### Decision
Chose **Supabase**.

### Trade-offs
| Pros | Cons |
|------|------|
| PostgreSQL for complex queries | Smaller ecosystem than Firebase |
| Row Level Security built-in | Fewer tutorials/examples |
| SQL knowledge transfers | Real-time requires extra setup |
| Better data modeling for relational data | |

### What I Would Improve
Next time, I'd evaluate the real-time requirements more carefully upfront. For heavily real-time apps, Firebase's realtime database might still be better despite the NoSQL limitations.

---

## 2. Multi-Provider LLM Fallback Architecture

**Date:** 2024

### Context
Building an AI chatbot that needed high reliability. Single LLM provider had occasional downtime and rate limits.

### Options Considered
1. **Single provider with retry** - Simple, but single point of failure
2. **Multi-provider with manual failover** - Reliable, but complex
3. **Multi-provider with automatic fallback** - Best UX, most complex

### Decision
Implemented **automatic fallback** with provider priority list.

### Trade-offs
| Pros | Cons |
|------|------|
| 99.9%+ effective uptime | Complex error handling |
| Graceful degradation | Need to normalize different API responses |
| Can optimize for cost vs speed | More API keys to manage |

### Implementation Details
```
Primary: OpenRouter (aggregates multiple models)
Fallback 1: Direct OpenAI API
Fallback 2: Gemini API
```

### What I Would Improve
Add better cost tracking per provider to make informed decisions about fallback order. Also add circuit breaker pattern to avoid hammering a failing provider.

---

## 3. Dual-Engine OCR Strategy

**Date:** 2024

### Context
Building an OCR service where accuracy was critical. Single OCR engine had varying quality on different document types.

### Options Considered
1. **Single engine (Tesseract)** - Simple, open source, widely documented
2. **Single engine (Paddle)** - Better accuracy, especially for Asian text
3. **Dual engine with confidence scoring** - Best accuracy, more complex

### Decision
Implemented **dual engine** approach with confidence-based selection.

### Trade-offs
| Pros | Cons |
|------|------|
| Best-of-both accuracy | Double processing time |
| Handles edge cases better | More infrastructure complexity |
| Can compare results for validation | Higher compute costs |

### What I Would Improve
Add ML-based routing to predict which engine will perform better for a given document, reducing unnecessary double processing.

---

## 4. React Query vs Redux for State Management

**Date:** 2024

### Context
New React application needed state management for server data and UI state.

### Options Considered
1. **Redux + RTK Query** - Full control, familiar, boilerplate heavy
2. **React Query + Context** - Server state handled well, simpler setup
3. **Zustand + React Query** - Lightweight, good separation

### Decision
Chose **React Query (TanStack Query)** for server state + React Context for UI state.

### Trade-offs
| Pros | Cons |
|------|------|
| Automatic caching and refetching | Learning curve for team |
| Optimistic updates built-in | Less control than Redux |
| DevTools excellent | Global UI state needs separate solution |

### What I Would Improve
For the next large project, I'd evaluate Zustand more seriously for global UI state instead of React Context, as it scales better.

---

## 5. Stripe Checkout vs Custom Payment Form

**Date:** 2024

### Context
SaaS product needed subscription payments. Had to decide between hosted Stripe Checkout and custom payment form.

### Options Considered
1. **Stripe Checkout (hosted)** - Fastest, handles compliance
2. **Stripe Elements (embedded)** - More control, still PCI-compliant
3. **Custom form** - Maximum control, PCI compliance nightmare

### Decision
Chose **Stripe Checkout (hosted)**.

### Trade-offs
| Pros | Cons |
|------|------|
| Zero PCI compliance burden | Less branding control |
| Mobile-optimized out of box | Users leave your site briefly |
| Handles edge cases (3DS, etc.) | Can't customize layout |

### What I Would Improve
Nothing - for an MVP/early-stage product, this was the right call. Would only consider Elements if branding requirements demanded it.

---

## 6. Flask vs FastAPI for Python Backend

**Date:** 2024

### Context
Building API backend for document processing service. Needed to choose Python web framework.

### Options Considered
1. **Flask** - Mature, flexible, lots of extensions
2. **FastAPI** - Modern, async, automatic docs
3. **Django REST** - Batteries included, might be overkill

### Decision
Chose **Flask** for this project.

### Trade-offs
| Pros | Cons |
|------|------|
| Team familiarity | No built-in async |
| Simple and flexible | Manual OpenAPI docs |
| Huge extension ecosystem | More boilerplate than FastAPI |

### What I Would Improve
For I/O-heavy services (lots of external API calls), would choose FastAPI next time for native async support. Flask's sync nature required careful worker configuration.

---

## 7. SQLite vs PostgreSQL for Local-First Tool

**Date:** 2024

### Context
Building a local-first media processing tool that needed persistent storage but might also sync to cloud.

### Options Considered
1. **SQLite** - Simple, file-based, no server needed
2. **PostgreSQL** - Powerful, but requires server
3. **JSON files** - Simplest, but no querying

### Decision
Chose **SQLite** with schema migrations.

### Trade-offs
| Pros | Cons |
|------|------|
| Zero configuration | Concurrent write limitations |
| Single file backup | No real-time subscriptions |
| SQL queries work | Need migration strategy |

### What I Would Improve
Would plan the sync-to-cloud story earlier. SQLite to PostgreSQL migration path needs careful planning.

---

## 8. Server Components vs Client Components (Next.js)

**Date:** 2024

### Context
Building Next.js 14+ application, needed to decide on component architecture.

### Options Considered
1. **Mostly Server Components** - Better performance, less JS shipped
2. **Mostly Client Components** - Familiar React patterns
3. **Hybrid approach** - Server for data, client for interactivity

### Decision
Adopted **hybrid approach** with Server Components as default.

### Trade-offs
| Pros | Cons |
|------|------|
| Smaller JS bundles | Need to think about boundaries |
| Direct database access in components | Can't use hooks in server components |
| Better SEO | State management more complex |

### What I Would Improve
Create clearer conventions upfront about which components should be client vs server. The "use client" boundary decisions compound.

---

## 9. Docker vs Direct Deployment

**Date:** 2024

### Context
Deploying Python application to cloud. Needed reproducible deployments.

### Options Considered
1. **Docker containers** - Consistent, portable
2. **Direct deployment** - Simpler, faster deploys
3. **Serverless (Lambda)** - Pay per use, cold starts

### Decision
Started with **Docker**, later added **Lambda** option.

### Trade-offs
| Pros | Cons |
|------|------|
| Works same everywhere | Learning curve |
| Easy local development | Larger deploy artifacts |
| Can run anywhere | Need container registry |

### What I Would Improve
Set up multi-architecture builds (ARM/x86) from the start. Hit issues deploying to ARM-based hosts later.

---

## 10. Rate Limiting Strategy

**Date:** 2024

### Context
API service being hit by occasional traffic spikes. Needed to protect backend and manage costs.

### Options Considered
1. **IP-based rate limiting** - Simple, but problematic for NAT
2. **User-based rate limiting** - Fair, requires auth
3. **Tiered rate limiting** - Different limits for different subscription tiers

### Decision
Implemented **tiered user-based rate limiting** with IP fallback for unauthenticated requests.

### Trade-offs
| Pros | Cons |
|------|------|
| Fair to paying customers | More complex implementation |
| Protects against abuse | Need to track usage per user |
| Upsell opportunity | Edge cases with IP + user overlap |

### What I Would Improve
Add better rate limit headers (X-RateLimit-Remaining etc.) from the start. Had to add them later when users requested visibility.

---

## 11. Vite vs Create React App

**Date:** 2024

### Context
Starting new React projects, CRA showing its age.

### Options Considered
1. **Create React App** - Familiar, but slow builds
2. **Vite** - Fast, modern, good DX
3. **Next.js** - Full framework, might be overkill

### Decision
Standardized on **Vite** for SPA projects.

### Trade-offs
| Pros | Cons |
|------|------|
| 10x faster HMR | Different config from CRA |
| Modern ES modules | Some CRA patterns don't transfer |
| Active development | |

### What I Would Improve
Nothing significant - Vite has been excellent. Would consider Next.js more for SEO-critical projects.

---

## 12. Playwright vs Puppeteer for Web Scraping

**Date:** 2024

### Context
Building web scraping tool for data collection. Needed headless browser automation.

### Options Considered
1. **Puppeteer** - Chrome-focused, mature
2. **Playwright** - Multi-browser, Microsoft-backed
3. **Selenium** - Old but stable

### Decision
Chose **Playwright**.

### Trade-offs
| Pros | Cons |
|------|------|
| Multi-browser support | Slightly larger install |
| Auto-wait is excellent | Different API from Puppeteer |
| Better debugging tools | |

### What I Would Improve
Would set up browser pool earlier for parallel scraping. Single browser instance became bottleneck.

---

## 13. Tailwind CSS vs CSS-in-JS

**Date:** 2024

### Context
Choosing styling approach for new React project.

### Options Considered
1. **Tailwind CSS** - Utility-first, consistent
2. **Styled Components** - Component-scoped, dynamic
3. **CSS Modules** - Simple, no runtime

### Decision
Adopted **Tailwind CSS** with component library (shadcn/ui).

### Trade-offs
| Pros | Cons |
|------|------|
| Rapid development | HTML can look cluttered |
| Consistent design tokens | Learning the utilities |
| No runtime CSS overhead | Bundle includes unused styles (purge helps) |

### What I Would Improve
Set up component extraction patterns earlier. Tailwind class lists get long and should be abstracted into components sooner.

---

## 14. Monorepo vs Multi-Repo

**Date:** 2024

### Context
Project grew to have frontend, backend, and shared utilities. Needed to organize code.

### Options Considered
1. **Monorepo** - Everything together, easy sharing
2. **Multi-repo** - Clear boundaries, independent deployment
3. **Hybrid** - Shared code in separate repo

### Decision
Chose **multi-repo** for separation, with shared types published as npm package.

### Trade-offs
| Pros | Cons |
|------|------|
| Clear ownership | Shared code changes are harder |
| Independent CI/CD | Need to coordinate releases |
| Smaller repo clones | Type synchronization friction |

### What I Would Improve
For a small team, would probably choose monorepo next time. The coordination overhead of multi-repo wasn't worth it at this scale.

---

## 15. Background Jobs: Redis Queue vs Database Queue

**Date:** 2024

### Context
Needed to process long-running tasks asynchronously.

### Options Considered
1. **Redis Queue (RQ/Bull)** - Fast, dedicated for jobs
2. **Database-backed queue** - Simpler, no extra infrastructure
3. **SQS/Cloud queues** - Managed, but vendor lock-in

### Decision
Chose **Redis Queue** (using Python RQ for Flask, Bull for Node).

### Trade-offs
| Pros | Cons |
|------|------|
| Very fast | Need Redis infrastructure |
| Built for this use case | Another thing to monitor |
| Good job visibility | Data not as durable as DB |

### What I Would Improve
Would evaluate managed solutions (SQS, Cloud Tasks) more seriously for production. Self-managing Redis adds operational burden.

---

## 16. Open-Source DMS vs Building Custom

**Date:** 2024

### Context
Client needed document management capabilities. Had to decide between using an established open-source solution or building custom.

### Options Considered
1. **Build custom** - Full control, but significant development time
2. **OpenKM** - Java-based, open-source, enterprise features
3. **SaaS solution** - Quick setup, but ongoing costs and data concerns

### Decision
Evaluated and deployed **OpenKM** (open-source DMS).

### Trade-offs
| Pros | Cons |
|------|------|
| No licensing costs | Requires Java/Tomcat expertise |
| Full data control | Self-hosted maintenance |
| Enterprise features out of box | Learning curve for customization |

### Implementation Notes
- Local deployment with Tomcat 9 and JDK 8
- Maven-based build for customizations
- WAR file deployment process

### What I Would Improve
Would document the deployment process better from the start. Java ecosystem has many configuration nuances that are easy to forget.

---

*This log is continuously updated as I make significant technical decisions.*
