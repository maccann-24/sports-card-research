# Sports Card Data Aggregation Platform - Executive Summary

**Research Completed:** January 29, 2026  
**Project Scope:** Build a comprehensive sports card pricing and collection tracking platform (CardLadder competitor)  
**Prepared For:** Development & Strategic Planning

---

## PROJECT OVERVIEW

### Mission
Build a best-in-class **sports card data aggregation platform** that provides collectors and investors with accurate, real-time pricing data, collection management tools, and market intelligence across multiple marketplaces and grading companies.

### Market Opportunity

**Sports Card Market Size:**
- **$13+ billion industry** (2025 estimate)
- Explosive growth since 2020 (COVID boom)
- Sustained interest from institutional investors, fractional ownership platforms
- Younger demographic entering hobby (Gen Z, Millennials)

**Current Market Leader:**
- **CardLadder.com** - $15/month Pro tier, 14+ marketplace coverage
- Strong brand but has weaknesses: pricing accuracy concerns, PSA-only populations, no public API

**Our Opportunity:**
- Better data coverage (more marketplaces, all grading companies)
- Superior pricing algorithms (ML-based predictions)
- Public API ecosystem (attract developers)
- International expansion (UK, Canada, Australia)
- B2B tools (card shops, dealers)

---

## TECHNICAL FEASIBILITY: ✅ HIGHLY VIABLE

### Core Data Sources

#### 1. eBay API (Primary Marketplace Data)
- **Status:** ✅ Production-ready APIs available
- **Coverage:** Millions of daily sports card listings
- **Rate Limits:** 5,000 calls/day free tier (scalable to 25K-100K+)
- **Data Quality:** Excellent (official API, structured data)
- **Cost:** FREE within rate limits

**Key APIs:**
- **Browse API:** Active listings, search, filters
- **Feed API:** Bulk downloads (75K calls/day)
- **Finding API:** Completed sales (⚠️ unreliable, use Feed API instead)

**Challenges:**
- Production access requires eBay Partner Network approval (2-4 weeks)
- Parsing unstructured titles (NLP required)
- Rate limits force prioritization (track top players/cards first)

#### 2. PSA API (Grading Data)
- **Status:** ✅ Public API available
- **Coverage:** Cert verification, card details, images
- **Rate Limits:** 100 calls/day free tier (paid tiers available)
- **Cost:** FREE (paid tiers pricing unknown - contact PSA)

**What You Get:**
- ✅ Certification verification (validate authenticity)
- ✅ Card details (player, year, set, grade)
- ✅ High-res images of graded slabs
- ❌ **NO population reports via API** (must scrape or manual entry)

**Critical Gap:** Population data is essential but not available via API. Workarounds:
- Web scraping PSA population report (legal risk, fragile)
- Manual data collection for top cards
- Partnership/licensing with PSA ($$$$)

#### 3. Additional Data Sources
- **PWCC Marketplace:** ❌ No API (scrape weekly auctions)
- **Goldin Auctions:** ❌ No API (scrape record sales)
- **Heritage Auctions:** ⚠️ Partner API available (requires business relationship)
- **BGS/SGC:** ❌ No APIs (cert verification via scraping)
- **Player Stats:** ✅ Multiple free/paid APIs (ESPN, SportsDataIO)

**Integration Priority:**
1. **Month 1-2:** eBay + PSA (MVP)
2. **Month 2-4:** PWCC, BGS/SGC cert extraction, player stats
3. **Month 4-6:** Goldin, Heritage, full BGS/SGC verification
4. **Month 6-12:** Fractional platforms, social sentiment

---

## DATABASE & ARCHITECTURE

### Recommended Tech Stack
- **PostgreSQL 16+** - Primary database (structured data)
- **TimescaleDB** - Time-series extension (pricing history)
- **Redis** - Caching layer (sub-millisecond lookups)
- **Elasticsearch** - Full-text search (card search)
- **AWS S3 / Cloudflare R2** - Image storage

### Data Scale (Year 1 Projections)
- **10 million+** cards in catalog
- **100 million+** historical sales records
- **50,000+** new listings processed daily
- **10,000+** active users
- **500GB-1TB** total data storage

### Schema Highlights
- **Players, Cards, Sets** - Master catalog (relational)
- **Sales, Active Listings** - Transactional data (partitioned by month)
- **Graded Cards** - Links cards to PSA/BGS/SGC cert data
- **Card Pricing** - Time-series snapshots (hourly/daily pricing)
- **PSA Populations** - Time-series population growth
- **User Collections** - Collection tracking with auto-valued portfolios

### ETL Pipeline
1. **Ingestion:** eBay API, PSA API, web scrapers → Redis queue
2. **Processing:** Parse titles (NLP), deduplicate, enrich with PSA data
3. **Storage:** PostgreSQL (structured), S3 (images)
4. **Pricing:** Hourly jobs calculate current values, trends
5. **Caching:** Redis (hot data), Elasticsearch (search index)

**Throughput:** Handle 50K+ listings/day with commodity hardware ($500-1,000/month infrastructure)

---

## COMPETITIVE ANALYSIS: CARDLADDER.COM

### What They Do Well
- ✅ **Comprehensive coverage** (14+ marketplaces)
- ✅ **Clean UX** (mobile apps, fast search)
- ✅ **Collection tracking** (sticky feature, daily engagement)
- ✅ **Brand recognition** (Sports Illustrated features, PSA partnership)
- ✅ **Research team** (manual verification of sales for accuracy)

### Their Weaknesses (Our Opportunities)
- ❌ **Pricing accuracy** (user complaints about values not matching recent comps)
- ❌ **PSA-only populations** (no BGS, SGC, CGC)
- ❌ **No public API** (closed ecosystem)
- ❌ **Limited free tier** (very restrictive, hard to attract users)
- ❌ **US-focused** (no international markets)

### Our Differentiation Strategy
1. **Better Pricing Algorithm**
   - ML-based predictions using player stats + market sentiment
   - Transparent methodology (show recent comps used)
   - Real-time updates (not daily batches)

2. **Multi-Grader Support**
   - PSA, BGS, SGC, CGC, Raw cards
   - BGS sub-grades (Centering, Corners, Edges, Surface)
   - Population tracking for all graders (not just PSA)

3. **Public API**
   - Freemium API tiers (free for hobbyists, paid for businesses)
   - Build ecosystem of third-party tools
   - Attract developers, create network effects

4. **Advanced Analytics**
   - Player performance correlation (stats → price prediction)
   - Market sentiment analysis (social media)
   - Portfolio optimization (risk analysis, diversification)

5. **International Expansion**
   - Multi-currency support (USD, GBP, EUR, CAD, AUD)
   - International marketplaces (eBay UK, Australia)
   - Global shipping/pricing considerations

6. **B2B Platform**
   - Tools for card shops (inventory management, POS integration)
   - Dealer dashboards
   - White-label solutions

---

## DEVELOPMENT ROADMAP

### Phase 1: MVP (Months 1-3)
**Goal:** Launch basic platform with core features

**Features:**
- Card database (top 500 players, major sets)
- eBay pricing data (active + historical via Feed API)
- PSA cert verification
- Basic collection tracking (add cards, track value)
- User accounts (free tier only)

**Tech Stack:**
- PostgreSQL + TimescaleDB
- Redis caching
- Basic web UI (React)
- eBay + PSA API integrations

**Team:**
- 2 backend engineers
- 1 frontend engineer
- 1 data engineer (ETL pipeline)

**Budget:** $50K-100K (salaries + infrastructure)

**Deliverable:** Working demo, 1,000+ beta users

---

### Phase 2: Enhanced Platform (Months 4-6)
**Goal:** Add differentiating features, launch Pro tier

**Features:**
- Expand card database (5,000+ players, all major sets)
- PWCC auction data (scraping)
- BGS/SGC cert extraction and verification
- Player stats integration (correlate performance with prices)
- Price prediction models (basic ML)
- Mobile apps (iOS + Android)
- Pro subscription tier ($10-15/month)

**Additional Data Sources:**
- PWCC weekly auctions
- Basic player stats (ESPN API)
- BGS/SGC cert verification (automated scraping)

**Team:**
- +1 ML engineer (pricing models)
- +1 mobile developer
- +1 researcher (manual data verification)

**Budget:** $150K-250K

**Deliverable:** Public launch, 10,000+ users, 500+ paying subscribers

---

### Phase 3: Growth & Scale (Months 7-12)
**Goal:** Establish market presence, scale to 50K+ users

**Features:**
- Comprehensive multi-grader support (PSA, BGS, SGC, CGC)
- Population report tracking (all graders, historical trends)
- Advanced analytics (predictive pricing, market sentiment)
- Public API (freemium tiers)
- International markets (UK, Canada, Australia)
- Social features (public profiles, following, comments)
- B2B tools (dealer dashboards, inventory management)

**Additional Data Sources:**
- Goldin, Heritage Auctions
- Social media sentiment (Twitter, Reddit)
- Fractional platforms (StarStock, Alt)
- Paid stats APIs (SportsDataIO)

**Team:**
- +2 engineers (API, international features)
- +2 researchers (expand verification team)
- +1 product manager
- +1 marketing/growth

**Budget:** $500K-750K (cumulative)

**Deliverable:** 50,000+ users, 2,500+ Pro subscribers, $30K-40K MRR

---

### Phase 4: Enterprise & Expansion (Year 2+)
**Goal:** Build moat, expand to B2B, dominate market

**Features:**
- Enterprise API tiers (high-volume access)
- White-label solutions for auction houses, card shops
- Advanced portfolio analytics (risk modeling, tax reporting)
- Insurance integrations (automatic coverage based on collection value)
- Physical card storage partnerships (PWCC Vault integration)
- Educational platform (courses, certifications for dealers)

**Team:** 20-30 employees (engineering, research, sales, marketing, operations)

**Budget:** $2M-5M/year

**Deliverable:** 200K+ users, 10K+ Pro subscribers, $150K+ MRR, B2B contracts with major players

---

## FINANCIAL PROJECTIONS

### Revenue Model

**Consumer (B2C):**
- **Free Tier:** Basic search, limited collection tracking, ads
- **Pro Tier ($15/month or $144/year):** 
  - All-time sales history
  - Advanced analytics
  - Price alerts
  - Unlimited collection size
  - Multi-marketplace search

**Business (B2B):**
- **API Access:** Freemium tiers
  - Free: 1,000 calls/month
  - Basic: $50/month (25K calls/month)
  - Pro: $500/month (500K calls/month)
  - Enterprise: Custom pricing
  
- **Data Licensing:** Sell aggregated market data to auction houses, dealers, investors
  
- **White Label:** Custom solutions for card shops, auction houses

**Advertising:**
- Card shops, grading services, insurance companies
- Sponsored listings

### Year 1 Projections
- **Users:** 0 → 50,000
- **Pro Subscribers:** 0 → 2,500 (5% conversion)
- **MRR:** $0 → $37,500 (2,500 × $15)
- **ARR:** $450K
- **API Revenue:** $5K-10K/month (Year 1 end)
- **Total Revenue (Year 1):** $250K-350K

### Year 2 Projections
- **Users:** 50K → 200K
- **Pro Subscribers:** 2,500 → 10,000 (5% conversion maintained)
- **MRR:** $150K
- **ARR (Subscriptions):** $1.8M
- **API + B2B Revenue:** $300K-500K
- **Total Revenue (Year 2):** $2M-2.5M

### Year 3 Projections
- **Users:** 200K → 500K
- **Pro Subscribers:** 10K → 25K
- **MRR:** $375K
- **ARR (Subscriptions):** $4.5M
- **API + B2B Revenue:** $1M-2M
- **Total Revenue (Year 3):** $5.5M-6.5M

**Path to Profitability:** Month 18-24 (with proper cost management)

---

## RISKS & MITIGATION

### Technical Risks

#### 1. eBay API Rate Limits / Access Loss
**Risk:** eBay restricts access or reduces rate limits  
**Likelihood:** Medium  
**Impact:** High (eBay is 70%+ of data)

**Mitigation:**
- Diversify data sources early (PWCC, Goldin, Heritage)
- Negotiate higher rate limits via Application Growth Check
- Build caching/historical database (reduce real-time dependency)
- eBay Partner Network compliance (follow all ToS)

#### 2. Web Scraping Blocks
**Risk:** Sites block scrapers (PWCC, Goldin, PSA populations)  
**Likelihood:** High  
**Impact:** Medium (affects data completeness)

**Mitigation:**
- Pursue official data partnerships
- Respectful scraping (rate limiting, user-agents)
- Manual data collection as fallback
- Community-contributed data (users submit sales)

#### 3. Database Scaling
**Risk:** Can't handle 100M+ records, slow queries  
**Likelihood:** Low (PostgreSQL scales well)  
**Impact:** High (user experience degrades)

**Mitigation:**
- Design for scale from day 1 (partitioning, indexing)
- TimescaleDB for time-series data
- Read replicas for query load distribution
- Aggressive caching (Redis)

### Business Risks

#### 4. CardLadder Competitive Response
**Risk:** CardLadder adds our differentiating features  
**Likelihood:** Medium-High  
**Impact:** High (reduces competitive advantage)

**Mitigation:**
- Move fast (launch MVP in 3 months)
- Build features they can't easily copy (proprietary ML models, partnerships)
- Focus on segments they ignore (B2B, international)
- Superior execution (better UX, faster updates)

#### 5. Market Downturn
**Risk:** Sports card market crashes (like 1990s)  
**Likelihood:** Medium (markets are cyclical)  
**Impact:** High (user engagement drops, churn increases)

**Mitigation:**
- Diversify revenue (not just Pro subscriptions)
- Build tools useful in bear markets (loss tracking, tax reporting)
- Lower pricing tier ($5-10/month option)
- Expand to related collectibles (comics, coins, etc.)

#### 6. Data Quality / Accuracy Concerns
**Risk:** Users don't trust our pricing data  
**Likelihood:** Medium (hard problem)  
**Impact:** High (reputation damage)

**Mitigation:**
- Hire manual verification team (like CardLadder)
- Transparent methodology (show recent comps)
- User feedback mechanism (report inaccurate prices)
- Regular audits vs actual sales

### Legal Risks

#### 7. Copyright / ToS Violations
**Risk:** Lawsuits from eBay, auction houses for scraping/data use  
**Likelihood:** Low-Medium  
**Impact:** High (cease & desist, platform shutdown)

**Mitigation:**
- Legal review of all scraping activities
- Pursue official partnerships where possible
- Fair use defense (aggregated facts, transformative use)
- Errors & omissions insurance
- Respect robots.txt and rate limits

#### 8. Trademark Issues
**Risk:** Using "PSA," "BGS," etc. without permission  
**Likelihood:** Low  
**Impact:** Medium (forced rebranding, legal fees)

**Mitigation:**
- Use only for factual reference (nominative fair use)
- Don't imply endorsement
- Legal counsel review of brand usage

---

## COST ANALYSIS

### Year 1 Costs

**Development (Months 1-12):**
- Engineering Team (4-6 engineers): $400K-600K
- Product Manager: $100K-150K
- Data Researchers (2): $80K-120K
- Total Salaries: **$580K-870K**

**Infrastructure:**
- Cloud hosting (AWS/GCP): $500-2,000/month
- Data APIs (eBay, PSA, stats): $100-500/month
- CDN, storage: $200-500/month
- Total Infrastructure (Year 1): **$10K-36K**

**Other Costs:**
- Legal (ToS, privacy, incorporation): $10K-20K
- Marketing (launch, ads): $20K-50K
- Office/tools: $20K-40K
- Total Other: **$50K-110K**

**Year 1 Total Costs:** **$640K-1.02M**

### Year 2 Costs
- Team expansion (10-15 people): $1M-1.5M
- Infrastructure (scaling): $50K-120K
- Marketing & sales: $100K-200K
- **Year 2 Total:** **$1.15M-1.82M**

### Funding Requirements
- **Seed Round (Pre-launch):** $750K-1M (runway to Month 12-15)
- **Series A (Growth):** $3M-5M (scale to 200K+ users, Year 2-3)

---

## GO / NO-GO RECOMMENDATION

### ✅ GO - This Project is Highly Viable

**Strengths:**
1. **Strong market opportunity** ($13B+ industry, growing)
2. **Technical feasibility** (APIs exist, data is accessible)
3. **Clear competitive advantage** (better data, multi-grader, API, ML)
4. **Defensible moat** (data network effects, partnerships, proprietary algorithms)
5. **Multiple revenue streams** (subscriptions, API, B2B, ads)
6. **Passionate target audience** (sports card collectors are engaged, willing to pay)

**Weaknesses to Address:**
1. **Capital intensive** (requires $750K-1M+ to reach scale)
2. **Execution risk** (CardLadder has head start, must move fast)
3. **Data dependencies** (eBay API restrictions, scraping fragility)
4. **Market risk** (sports card market could cool off)

**Critical Success Factors:**
1. **Nail pricing accuracy** (this is THE product)
2. **Move fast** (MVP in 3 months, public launch in 6 months)
3. **Build partnerships** (PSA, auction houses for official data)
4. **Superior UX** (mobile-first, fast, intuitive)
5. **Strong team** (experienced engineers, data scientists, sports card experts)

---

## RECOMMENDED NEXT STEPS

### Immediate (Week 1-2)
1. **Validate market demand**
   - Interview 20-30 sports card collectors
   - Survey CardLadder users (what's missing?)
   - Join sports card forums, Discord, Facebook groups

2. **Technical validation**
   - Build eBay API proof-of-concept (fetch 1,000 listings)
   - Build PSA API proof-of-concept (verify 100 certs)
   - Test pricing algorithm on sample data (1 month of sales for 1 card)

3. **Assemble core team**
   - Technical co-founder / lead engineer
   - Sports card domain expert (former dealer/collector)
   - Product manager (or founder as PM initially)

### Short-Term (Month 1-3)
1. **Legal & business setup**
   - Incorporate (Delaware C-Corp or LLC)
   - Draft terms of service, privacy policy
   - eBay Partner Network application
   - PSA API account setup

2. **Build MVP**
   - Core database schema (PostgreSQL)
   - eBay integration (Browse + Feed APIs)
   - PSA integration (cert verification)
   - Basic web UI (search, collection tracking)
   - 100 beta users

3. **Secure funding**
   - Angel investors (sports card collectors as angels?)
   - Pre-seed or seed round ($500K-1M)

### Medium-Term (Month 4-6)
1. **Public launch**
   - Press release, media outreach
   - Reddit, Twitter, Discord promotion
   - Launch Pro tier ($15/month)
   - Target: 10,000 users, 500 Pro subscribers

2. **Expand data sources**
   - PWCC scraping
   - BGS/SGC cert extraction
   - Player stats integration

3. **Mobile apps**
   - iOS app (TestFlight beta)
   - Android app (Google Play beta)

---

## APPENDICES

### Document Index
1. **01-EBAY-API.md** - Complete eBay API documentation, authentication, endpoints, code examples
2. **02-PSA-API.md** - PSA API guide, population report workarounds, cert verification
3. **03-CARDLADDER-ANALYSIS.md** - Competitive analysis, features, business model, weaknesses
4. **04-DATABASE-ARCHITECTURE.md** - Complete schema, ETL pipeline, scaling strategy
5. **05-ADDITIONAL-DATA-SOURCES.md** - PWCC, Goldin, player stats, BGS/SGC, social media

### Key Metrics to Track
- **User Growth:** Daily active users (DAU), monthly active users (MAU)
- **Engagement:** Cards searched/day, collections created, daily logins
- **Conversion:** Free → Pro conversion rate (target: 5%)
- **Retention:** Monthly churn rate (target: <5%)
- **Revenue:** MRR, ARR, LTV/CAC ratio
- **Data Quality:** Pricing accuracy (vs actual sales), data completeness (% of market covered)

### Success Milestones
- **Month 3:** MVP launched, 1,000 beta users
- **Month 6:** Public launch, 10,000 users, $5K MRR
- **Month 12:** 50,000 users, $30K MRR, break-even trajectory
- **Month 18:** 100,000 users, $75K MRR, profitability
- **Year 3:** 500,000 users, $375K MRR, market leadership

---

## CONCLUSION

Building a sports card data aggregation platform is **technically feasible, commercially viable, and strategically sound**. The market opportunity is massive ($13B+ industry), the data sources are accessible (eBay + PSA APIs work, scraping is possible), and there's a clear path to differentiation vs CardLadder (better pricing, multi-grader support, public API, ML predictions).

**The window is open NOW.** The sports card market is hot, CardLadder has proven demand for this product, and there's room for a superior platform with better execution.

**Critical dependencies:**
- Secure funding ($750K-1M seed round)
- Hire experienced engineering + data science team
- Move FAST (MVP in 3 months, launch in 6 months)
- Build partnerships (PSA, auction houses) for data reliability

**If executed well, this platform can achieve:**
- 500K+ users in Year 3
- $4-5M+ ARR
- Market leadership position
- Strategic acquisition target ($50M-100M+ valuation)

**Recommendation: PROCEED with full development. The opportunity is real and the timing is right.**

---

**Research Completed by:** Claude (Anthropic)  
**Date:** January 29, 2026  
**Contact:** See detailed documentation in companion files

🚀 **Time to build.**
