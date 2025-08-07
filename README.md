# Bangladesh Accident News Scraper - Architecture

## System Overview

```
                            ┌─────────────────────────────────┐
                            │         SCHEDULER               │
                            │    (Cron + Cost Control)        │
                            └─────────────────┬───────────────┘
                                              │
      ┌───────────────────────────────────────┼───────────────────────────────────────┐
      │                                       │                                       │
      │    ┌─────────────┐     ┌─────────────┐│┌─────────────┐     ┌─────────────┐    │
      │    │   SCRAPER   │────▶     FILTER  ────▶ LLM API   ────▶  STORAGE    │    │
      │    │  8 Papers   │     │ (Keywords)  │ │ (Batch)     │     │ PostgreSQL  │    │
      │    │ Parallel    │     │ 90% Drop    │ │ GPT-4 Mini  │     │   + JSON    │    │
      │    └─────────────┘     └─────────────┘ └─────────────┘     └─────────────┘    │
      │                                                                               │
      └───────────────────────────────────────────────────────────────────────────────┘
                               
```

## Core Components

### 1. **Task Scheduler**
- **Daily Cron Job** 
- **Redis Queue** for task management

### 2. **Web Scraping Engine**
- **Target Sources**: All possible Bengali newspapers
- **Scrapy Framework** with custom spiders
- **Accident Keyword Filter** (pre-LLM screening)

### 3. **LLM Processing Pipeline**
- **OpenAI GPT-4 Mini** or **Gemini Flash**
- **Structured JSON output** with confidence scores
- **Batch processing** (Specified amount of articles per API call)

### 4. **Data Storage**
- **PostgreSQL** for data store
- **Automated backups** and cleanup

## Target News Sources
1. **Prothom Alo** (prothomalo.com)
2. **The Daily Star** (thedailystar.net) 
3. **Bangladesh Pratidin** (bd-pratidin.com)
4. **Jugantor** (jugantor.com)
5. **Ittefaq** (ittefaq.com.bd)
6. **New Age** (newagebd.net)
7. **Dhaka Tribune** (dhakatribune.com)
8. **UNB News** (unb.com.bd)
etc.


## Detailed Daily Workflow

### **Phase 1: Initialization**
```
 - Cron triggers main scheduler
 - Health check all services (Redis, PostgreSQL, API keys)
 - Load yesterday's processed URLs from cache
 - Initialize worker pools (4 scrapers + 2 LLM processors)
 - Set daily cost limits and counters
```

### **Phase 2: Multi-Source Scraping**
```
Parallel Scraping Strategy:
├── Worker 1: Prothom Alo + Daily Star (English/Bengali mix)
├── Worker 2: Bangladesh Pratidin + Jugantor (Bengali focus)  
├── Worker 3: Ittefaq + New Age (Regional coverage)
└── Worker 4: Dhaka Tribune + UNB (Breaking news + wire)

 - Start parallel scraping with staggered delays
 - Scrape homepage + crime/local sections
 - Follow pagination for last 24 hours
 - Extract article URLs and basic metadata
 - Download full article content
 - Apply first-level accident detection filter
 - Store filtered articles in Redis processing queue
 - Complete scraping phase (estimated 800-1200 articles)
```

### **Phase 3: Intelligent Filtering**
```
- Load articles from Redis queue
- Apply enhanced Bengali/English keyword filter:
          Bengali: দুর্ঘটনা, মৃত্যু, আহত, সড়ক দুর্ঘটনা, ট্রেন দুর্ঘটনা
          English: accident, death, injured, killed, casualty, collision
- Content length filtering (min 100 words)
- Location keyword filtering (Bangladesh districts/cities)
- Duplicate detection using content similarity
- Final queue: ~80-120 accident-related articles (90% reduction)
- Queue articles for LLM processing
```

### **Phase 4: LLM Processing**
```
- Batch articles into groups of 5
- Start parallel LLM processing (2 workers)
├── Worker A: Process batches 1, 3, 5... (OpenAI GPT-4 Mini)
└── Worker B: Process batches 2, 4, 6... (Gemini Flash backup)

- Extract structured data for each article
- Validate extracted data (location, casualty numbers)
- Apply confidence scoring (keep score > 0.6)
- Cache results in Redis for 30 days
- Complete LLM processing (~60-90 verified accidents)
```

### **Phase 5: Data Storage & Validation**
```
 - Load processed data from Redis
 - Geographic validation (match with BD districts/divisions)
 - Casualty number reasonableness check
 - Cross-reference duplicate incidents across sources
 - Merge similar incidents (same location + time)
 - Insert unique incidents into PostgreSQL
 - Update daily statistics
 - Generate incident IDs and relationships
 - Complete data storage
```

### **Phase 6: Reporting**
```
 - Generate daily summary statistics
 - Complete daily workflow
```


