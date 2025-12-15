# Case Studies

Anonymized case studies from production systems I've built. Client names, specific metrics, and proprietary details have been redacted.

---

## Case Study 1: Building a Reliable AI Chatbot Service

### Problem

A client needed a ChatGPT-like interface for their users, but with specific requirements:
- Must work 99.9% of the time (single LLM provider wasn't reliable enough)
- Must handle the LLM occasionally refusing to answer certain questions
- Must support multiple subscription tiers with different usage limits
- Must feel responsive despite LLM latency

### Constraints

- Budget limited which LLM providers we could use
- Existing user base expected immediate availability
- Team had limited experience with LLM APIs
- No tolerance for visible errors to end users

### Approach

**1. Multi-Provider Architecture**

Built an LLM router that automatically fails over between providers:

```
Request → Primary Provider (OpenRouter)
              ↓ (on error/timeout)
          Fallback 1 (Direct OpenAI)
              ↓ (on error/timeout)
          Fallback 2 (Gemini)
              ↓ (all failed)
          Graceful error message
```

**2. Refusal Detection**

Implemented pattern matching to detect when an LLM refuses to answer:
- "I cannot assist with..."
- "I'm not able to..."
- [Other patterns]

When detected, automatically retry with a different provider or rephrased prompt.

**3. Streaming Response UX**

Implemented ChatGPT-style streaming:
- Show response character by character
- Allow user to stop generation mid-stream
- Smooth scrolling to follow new content

**4. Tiered Rate Limiting**

```
Free tier: 10 messages/day
Basic tier: 100 messages/day
Pro tier: Unlimited
```

Rate limits checked server-side with Redis, clear UI feedback when approaching limits.

### Key Technical Choices

| Decision | Why |
|----------|-----|
| OpenRouter as primary | Aggregates multiple models, good reliability |
| Redis for rate limiting | Fast, atomic operations |
| Server-Sent Events for streaming | Better browser support than WebSockets |
| Supabase for user data | Easy auth + PostgreSQL |

### Outcome

- Achieved effective 99.95% uptime (vs ~98% with single provider)
- Refusal rate dropped from ~5% to <1% with retry logic
- User engagement increased due to responsive streaming UX
- Successfully launched with tiered subscriptions

### Lessons Learned

1. **LLM APIs are unreliable** - Always plan for failures
2. **Streaming improves perceived performance** significantly
3. **Rate limiting UX matters** - Users accept limits if communicated clearly
4. **Provider diversity is worth the complexity** for critical paths

---

## Case Study 2: Multi-Engine OCR System

### Problem

Building an OCR service for document digitization. Single OCR engine had inconsistent quality:
- Good for printed English text
- Poor for handwriting
- Struggling with Asian characters
- Variable performance on low-quality scans

### Constraints

- Processing time needed to be under 30 seconds
- Accuracy was more important than speed
- Budget for cloud OCR APIs was limited
- Needed to handle 1000+ documents per day

### Approach

**1. Dual-Engine Processing**

Run two OCR engines in parallel:
- **Engine A (Paddle OCR)**: Better for Asian text, handwriting
- **Engine B (Tesseract)**: Better for standard printed documents

**2. Confidence-Based Selection**

Each engine provides confidence scores. Selection logic:

```python
# Pseudocode
if engineA.confidence > 0.9:
    use engineA result
elif engineB.confidence > 0.9:
    use engineB result
elif engineA.confidence > engineB.confidence:
    use engineA result
else:
    use engineB result
```

**3. AI Post-Processing**

Raw OCR output often has issues. Added LLM cleanup:
- Fix obvious spelling errors
- Restore paragraph structure
- Format tables and lists

**4. Image Preprocessing**

Before OCR:
- Auto-rotate detection
- Contrast enhancement
- Noise reduction
- Deskewing

### Key Technical Choices

| Decision | Why |
|----------|-----|
| Python Flask backend | Team familiarity, good OCR library support |
| Parallel engine processing | Better accuracy without doubling time |
| AWS Lambda for scaling | Handle burst traffic without always-on costs |
| SQLite for job tracking | Simple, no extra infrastructure |

### Outcome

- Accuracy improved from ~85% to ~95% compared to single engine
- Processing time averaged 15 seconds (within requirement)
- Handled traffic spikes during business hours smoothly
- Reduced manual correction work significantly

### Lessons Learned

1. **Multiple engines beat single engine** for diverse document types
2. **Preprocessing is crucial** - garbage in, garbage out
3. **Confidence scores aren't always reliable** - needed manual tuning
4. **AI post-processing adds significant value** for final quality

---

## Case Study 3: Educational Quiz Platform

### Problem

Building a test preparation platform for a standardized exam. Requirements:
- Thousands of practice questions with explanations
- Track user progress across sessions
- Adapt difficulty based on performance
- Subscription-based monetization

### Constraints

- Launch timeline was tight (2 months)
- Content needed to be scraped from various sources legally
- Mobile experience was critical (80%+ mobile users expected)
- Limited budget for infrastructure

### Approach

**1. Content Pipeline**

```
Source Sites → Playwright Scraper → Content Processor → Review Queue → Database
```

Used Playwright for scraping, with:
- Rate limiting to be respectful
- Error recovery for network issues
- Duplicate detection

**2. Quiz Engine**

Adaptive question selection:
- Track correct/incorrect per topic
- Weight questions by weakness areas
- Spaced repetition for retention

**3. Progress Tracking**

```
UserProgress {
  topicScores: Map<TopicID, Score>
  questionsAttempted: number
  streakDays: number
  lastActive: timestamp
}
```

Synced to Firestore for cross-device access.

**4. Subscription System**

- Free tier: Limited questions per day
- Premium: All questions + explanations + progress tracking
- Stripe Checkout for payments
- Webhook handlers for subscription events

### Key Technical Choices

| Decision | Why |
|----------|-----|
| Next.js + Firebase | Fast development, good mobile perf |
| Firestore | Real-time sync, offline support |
| Stripe Checkout | Fastest path to payments |
| Playwright for scraping | Handles JS-heavy sites |

### Outcome

- Launched within 2-month timeline
- Mobile-first design achieved good engagement
- Conversion rate to premium exceeded expectations
- Content pipeline processed 5000+ questions

### Lessons Learned

1. **Scraping is fragile** - sites change, need monitoring
2. **Firebase costs can spike** - optimize reads/writes early
3. **Gamification drives engagement** - streaks, scores matter
4. **Mobile-first is worth the effort** for consumer apps

---

## Case Study 4: Media Processing Pipeline

### Problem

Building a tool to process YouTube videos and generate useful outputs:
- Download subtitles/transcripts
- Generate AI summaries
- Create audio versions (TTS)
- Support multiple languages

### Constraints

- YouTube's API has strict rate limits
- TTS APIs are expensive at scale
- Processing time can be long, need async handling
- Users expect real-time progress updates

### Approach

**1. Async Task Architecture**

```
User Request → Create Task → Queue Job → Background Worker → Store Result
                    ↓
              Return Task ID
                    ↓
              Poll for Status
```

**2. Multi-Source Subtitle Fetching**

YouTube subtitle availability varies. Implemented fallback:
1. Try official subtitles API
2. Try auto-generated captions
3. Fall back to Whisper transcription (costly, last resort)

**3. TTS with Voice Selection**

Supported multiple TTS providers:
- AWS Polly (English, many voices)
- Azure Speech (Better for Asian languages)
- Voice selection UI for user preference

**4. Progress Tracking**

Real-time progress via polling:
```json
{
  "taskId": "abc123",
  "status": "processing",
  "progress": 65,
  "currentStep": "generating_audio",
  "estimatedTimeRemaining": 45
}
```

### Key Technical Choices

| Decision | Why |
|----------|-----|
| Flask + Redis Queue | Good for long-running Python tasks |
| Polling over WebSockets | Simpler, works with CDNs |
| Multi-provider TTS | Different providers excel at different languages |
| SQLite for local storage | Simple deployment, easy backup |

### Outcome

- Processed thousands of videos reliably
- Users appreciated progress visibility
- Multi-language TTS expanded market
- Async architecture handled variable processing times

### Lessons Learned

1. **YouTube APIs change frequently** - need monitoring and fallbacks
2. **TTS costs add up** - caching and deduplication critical
3. **Progress feedback is essential** for long tasks
4. **Background processing simplifies UX** significantly

---

## Case Study 5: Real-Time Domain Search Tool

### Problem

Building a domain name search tool with AI suggestions:
- Generate creative domain names from keywords
- Check availability in real-time
- Provide pricing information
- Support bulk checking

### Constraints

- Domain availability APIs have rate limits
- LLM suggestions need to be fast
- Users expect instant feedback
- Many TLDs to check

### Approach

**1. AI Name Generation**

Prompt engineering for creative suggestions:
- Input: "coffee shop in brooklyn"
- Output: brooklynbrew.com, coffeebrk.co, bkcoffee.shop, etc.

**2. Parallel Availability Checking**

```
Generated Names → Parallel API Calls → Aggregate Results
                     ↓
              Check .com, .co, .io, .net simultaneously
```

**3. Caching Layer**

Domain availability doesn't change often. Cache results:
- 15-minute cache for availability
- 24-hour cache for pricing
- Reduces API costs significantly

**4. Progressive Loading**

Show results as they come in:
1. Show generated names immediately
2. Update availability as checks complete
3. Add pricing once available

### Key Technical Choices

| Decision | Why |
|----------|-----|
| Next.js 15 | Server components for SEO |
| React 19 | Latest features, good performance |
| Edge functions | Low latency for availability checks |
| Streaming responses | Progressive result display |

### Outcome

- Sub-second name generation
- Availability checks complete in 2-3 seconds
- Reduced API costs 70% with caching
- Good user feedback on speed

### Lessons Learned

1. **Parallel requests are essential** for bulk operations
2. **Caching is crucial** for rate-limited APIs
3. **Progressive loading improves perceived speed** dramatically
4. **AI prompts need iteration** for quality suggestions

---

*These case studies represent real projects with details anonymized for confidentiality.*
