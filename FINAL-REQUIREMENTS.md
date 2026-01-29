# Final Project Requirements

**Date:** January 29, 2026  
**Status:** Ready to Build

---

## PROJECT SCOPE

### Data Collection
- **Sports:** All (Football, Basketball, Baseball, Hockey, Soccer, etc.)
- **Price Range:** All (no minimum threshold)
- **Lookback Window:** 120 days (4 months)
- **Focus:** PSA-graded cards only

### Data Sources
1. **eBay API:** Sale price, sale date, listing type (AUCTION/FIXED_PRICE)
2. **PSA Population Reports:** PSA 10/9/8 counts (scraped)

### Matching Strategy
- Year + Set + Card Number must match exactly
- Player name fuzzy match (0.85+ similarity)
- PostgreSQL trigram similarity for speed

---

## KEY FEATURES

### 1. Time-Weighted Pricing (120-day window)
```sql
-- Exponential decay: 50% weight every 90 days
-- But only consider sales from last 120 days
SELECT 
  SUM(sale_price * EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400))) / 
  SUM(EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400)))
FROM ebay_sales
WHERE card_id = $1
  AND grader = 'PSA'
  AND grade = '10'
  AND sale_date > NOW() - INTERVAL '120 days';
```

### 2. Condition Sensitivity
- Calculate PSA 10 premium (PSA 10 price / PSA 9 price)
- Low (1.0-1.3x), Medium (1.3-2.0x), High (2.0x+)

### 3. Filterable Opportunities
- Filter by: Sport, Set, Year
- Sort by discount percentage
- Show PSA population counts

---

## DATABASE SCHEMA (Simplified)

```sql
-- Cards catalog
CREATE TABLE cards (
  id SERIAL PRIMARY KEY,
  player_name VARCHAR(200) NOT NULL,
  year VARCHAR(10),
  manufacturer VARCHAR(100),
  set_name VARCHAR(200),
  card_number VARCHAR(50),
  sport VARCHAR(50),
  
  -- Normalized for matching
  normalized_name VARCHAR(200),
  
  -- Full-text search
  search_vector tsvector GENERATED ALWAYS AS (
    to_tsvector('english', 
      COALESCE(player_name, '') || ' ' ||
      COALESCE(year, '') || ' ' ||
      COALESCE(set_name, '')
    )
  ) STORED,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  INDEX idx_sport (sport),
  INDEX idx_year_set (year, set_name),
  INDEX idx_search (search_vector) USING GIN
);

-- eBay sales
CREATE TABLE ebay_sales (
  id SERIAL PRIMARY KEY,
  card_id INT REFERENCES cards(id),
  item_id VARCHAR(100) UNIQUE,
  title TEXT,
  
  -- Core data
  sale_price DECIMAL(10,2) NOT NULL,
  sale_date TIMESTAMP NOT NULL,
  listing_type VARCHAR(20),  -- 'AUCTION' or 'FIXED_PRICE'
  
  -- PSA grading
  graded BOOLEAN DEFAULT true,
  grade VARCHAR(10),
  grader VARCHAR(50) CHECK (grader = 'PSA'),
  
  -- Matching metadata
  match_confidence DECIMAL(3,2),
  needs_review BOOLEAN DEFAULT FALSE,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  INDEX idx_card_sales (card_id, sale_date),
  INDEX idx_recent_sales (sale_date) WHERE sale_date > NOW() - INTERVAL '120 days'
);

-- PSA populations
CREATE TABLE psa_populations (
  id SERIAL PRIMARY KEY,
  card_id INT REFERENCES cards(id),
  
  psa_10_count INT,
  psa_9_count INT,
  psa_8_count INT,
  total_graded INT,
  
  snapshot_date DATE DEFAULT CURRENT_DATE,
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE (card_id, snapshot_date)
);

-- Install extensions
CREATE EXTENSION IF NOT EXISTS pg_trgm;  -- Trigram similarity
```

---

## IMPLEMENTATION PHASES

### Phase 1: eBay Scraper (Week 1)
**Goal:** Collect PSA-graded sales from eBay

**Code:** `scrapers/ebay-scraper.js`

**What it does:**
1. Search eBay for "{player} {year} {set} PSA"
2. Filter results (only PSA in title)
3. Extract: price, date, listing type
4. Parse: player, year, set, card number, grade
5. Store in database (create card if new, link sale to card)

**Target:** 1,000-5,000 sales/day (within free API limits)

### Phase 2: PSA Population Scraper (Week 2)
**Goal:** Scrape PSA population reports

**Code:** `scrapers/psa-population-scraper.js`

**What it does:**
1. Identify top cards from eBay sales (most activity)
2. Search PSA website for population reports
3. Extract PSA 10/9/8 counts
4. Match to cards in database
5. Store population snapshots

**Target:** 100-500 cards/day (manual priority list)

### Phase 3: Matching & Analysis (Week 3)
**Goal:** Link eBay sales to PSA populations

**Code:** `scripts/match-cards.js` + SQL views

**What it does:**
1. Calculate time-weighted prices (120 days)
2. Calculate PSA 10 premiums (condition sensitivity)
3. Find arbitrage opportunities (raw vs PSA 10 discount)
4. Generate reports

### Phase 4: Query Interface (Week 4)
**Goal:** Simple UI to explore opportunities

**Options:**
- **SQL queries** (fastest, use pgAdmin)
- **Web UI** (Next.js + Vercel, 1-2 days extra)
- **Google Sheets** (Apps Script queries)

---

## EXAMPLE QUERIES

### Find Arbitrage Opportunities (All Sports)
```sql
WITH recent_sales AS (
  SELECT 
    card_id,
    graded,
    grade,
    -- Time-weighted average (120 days only)
    SUM(sale_price * EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400))) / 
    SUM(EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400))) as weighted_avg_price,
    COUNT(*) as sale_count,
    MAX(sale_date) as last_sale_date
  FROM ebay_sales
  WHERE sale_date > NOW() - INTERVAL '120 days'
    AND grader = 'PSA'
  GROUP BY card_id, graded, grade
)

SELECT 
  c.sport,
  c.player_name,
  c.year,
  c.set_name,
  c.card_number,
  
  -- PSA 10 pricing
  psa10.weighted_avg_price as psa10_avg,
  psa10.sale_count as psa10_sales,
  
  -- Raw pricing
  raw.weighted_avg_price as raw_avg,
  raw.sale_count as raw_sales,
  
  -- Discount
  ROUND(100 * (1 - raw.weighted_avg_price / psa10.weighted_avg_price), 1) as discount_pct,
  
  -- PSA population
  pop.psa_10_count,
  pop.psa_9_count,
  ROUND(pop.psa_10_count::numeric / NULLIF(pop.psa_9_count, 0), 2) as psa10_to_psa9_ratio,
  
  -- Freshness
  EXTRACT(DAY FROM NOW() - psa10.last_sale_date) as days_since_last_psa10_sale

FROM cards c

-- PSA 10 sales
LEFT JOIN recent_sales psa10 ON psa10.card_id = c.id AND psa10.graded = true AND psa10.grade = '10'

-- Raw sales
LEFT JOIN recent_sales raw ON raw.card_id = c.id AND raw.graded = false

-- PSA population
LEFT JOIN psa_populations pop ON pop.card_id = c.id AND pop.snapshot_date = CURRENT_DATE

WHERE 
  -- Require minimum data
  psa10.sale_count >= 3
  AND raw.sale_count >= 2
  
  -- Require meaningful discount
  AND (1 - raw.weighted_avg_price / psa10.weighted_avg_price) >= 0.20  -- 20%+
  
  -- Exclude stale data
  AND psa10.last_sale_date > NOW() - INTERVAL '30 days'

ORDER BY 
  discount_pct DESC
LIMIT 100;
```

### Filter by Sport/Set/Year
```sql
-- Same query as above, add WHERE clauses:
WHERE 
  c.sport = 'Football'  -- Filter by sport
  AND c.year = '2023'   -- Filter by year
  AND c.set_name = 'Prizm'  -- Filter by set
  -- ... rest of conditions
```

### Condition Sensitivity Analysis
```sql
SELECT 
  c.sport,
  c.player_name,
  c.year,
  c.set_name,
  
  -- PSA 10 vs PSA 9 pricing
  psa10.weighted_avg_price as psa10_avg,
  psa9.weighted_avg_price as psa9_avg,
  
  -- Premium ratio
  ROUND(psa10.weighted_avg_price / NULLIF(psa9.weighted_avg_price, 0), 2) as premium_ratio,
  
  -- Classification
  CASE 
    WHEN psa10.weighted_avg_price / NULLIF(psa9.weighted_avg_price, 0) < 1.3 THEN 'Low'
    WHEN psa10.weighted_avg_price / NULLIF(psa9.weighted_avg_price, 0) < 2.0 THEN 'Medium'
    ELSE 'High'
  END as sensitivity

FROM cards c
LEFT JOIN recent_sales psa10 ON psa10.card_id = c.id AND psa10.grade = '10'
LEFT JOIN recent_sales psa9 ON psa9.card_id = c.id AND psa9.grade = '9'

WHERE psa10.sale_count >= 3 AND psa9.sale_count >= 3

ORDER BY premium_ratio DESC;
```

---

## COST ESTIMATE (All Sports, All Prices)

### Infrastructure
- **Supabase Pro:** $25/month (8GB database, enough for 100K+ sales)
- **Railway scrapers:** $10/month (1GB RAM, always-on)
- **Cloudflare R2 (optional images):** $2/month (50GB)
- **Total:** $37/month

### Data Volume (Estimated)
- **Cards tracked:** 10,000-50,000 (all sports, all price ranges)
- **Sales/day:** 5,000-10,000 (eBay API limit: 5K calls/day)
- **Database size:** 2-5GB (Year 1)

### API Limits
- **eBay:** 5,000 calls/day (FREE) → ~1,000-2,000 cards searched/day
- **PSA:** 100 calls/day (FREE) → focus on high-activity cards

---

## STARTER CODE STRUCTURE

```
sports-card-research/
├── README.md
├── REQUIREMENTS.md
├── DATA-MATCHING.md
│
├── db/
│   ├── schema.sql           # Database schema
│   ├── functions.sql        # Time-weighted pricing functions
│   └── queries.sql          # Pre-built opportunity finder queries
│
├── scrapers/
│   ├── ebay-scraper.js      # eBay API integration
│   ├── psa-population-scraper.js  # PSA population scraper
│   └── matching.js          # Card matching logic
│
├── scripts/
│   ├── setup-database.js    # Initialize Supabase
│   ├── test-ebay-api.js     # Test eBay connection
│   └── analyze-opportunities.js  # Run opportunity finder
│
├── config/
│   ├── .env.example         # API keys template
│   └── players.js           # Priority player list
│
└── package.json
```

---

## NEXT STEPS

1. **Set up Supabase** (5 minutes)
   - Create account: https://supabase.com
   - Create project (free tier)
   - Save connection string

2. **Apply for eBay API keys** (2-4 weeks)
   - Sign up: https://developer.ebay.com
   - Get sandbox keys (immediate testing)
   - Apply for production keys (2-4 week approval)

3. **Get PSA API key** (instant)
   - Sign up: https://www.psacard.com/publicapi
   - Get API key (free, immediate)

4. **Clone starter repo**
   - I'll create `sports-card-starter` with all code
   - You run `npm install`, configure `.env`, start scrapers

---

## DELIVERABLES (4 Weeks)

**End of Week 1:**
- eBay scraper running (1,000+ sales collected)
- Database with cards + sales tables populated

**End of Week 2:**
- PSA population scraper running (100+ cards with population data)
- Card matching logic working (90%+ accuracy)

**End of Week 3:**
- Time-weighted pricing functions
- Condition sensitivity analysis
- Opportunity finder query tested

**End of Week 4:**
- Query interface (pgAdmin or simple web UI)
- Daily automated scraper jobs
- 5,000-10,000 sales in database

---

## QUESTIONS ANSWERED

1. **Sports?** ✅ All sports
2. **Price range?** ✅ All prices (no minimum)
3. **Lookback window?** ✅ 120 days
4. **Time decay?** ✅ 90-day half-life (can tune)
5. **Risk tolerance?** ✅ Show all opportunities (user filters by sensitivity)

---

**Ready to build. Shall I create the starter code repository?**
