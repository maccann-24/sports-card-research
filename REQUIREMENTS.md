# Project Requirements (Updated with Partner Feedback)

**Date:** January 29, 2026  
**Contributors:** Matthew + Partner

---

## CORE STRATEGY

### Focus: PSA-Only Arbitrage Opportunities

**Goal:** Find raw cards selling below PSA 10 market value, accounting for:
1. Time gaps between sales (player performance changes)
2. Condition sensitivity (high PSA 10 premiums indicate raw cards are typically damaged)
3. Sport/set/year specificity (not just catch-all opportunities)

---

## KEY REQUIREMENTS

### 1. Time-Weighted Pricing Algorithm

**Problem:** A raw card selling at a "discount" vs PSA 10 might be explained by:
- Long time gap (player got better/worse)
- Market shift (card got hot/cold)
- Seasonal effects (playoffs, off-season)

**Solution:** Apply time decay penalty to old sales

**Formula:**
```sql
-- Weighted average with time decay
CREATE FUNCTION calculate_current_price(card_id INT) RETURNS DECIMAL AS $$
  SELECT 
    SUM(sale_price * time_weight) / SUM(time_weight)
  FROM (
    SELECT 
      sale_price,
      -- Exponential decay: sales lose 50% weight every 90 days
      EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400)) as time_weight
    FROM ebay_sales
    WHERE card_id = $1
      AND sale_date > NOW() - INTERVAL '1 year'  -- Only last year
      AND graded = true
      AND grade = '10'
      AND grader = 'PSA'
  ) weighted_sales;
$$ LANGUAGE SQL;
```

**Parameters to tune:**
- **Half-life:** 90 days (adjust based on sport volatility)
- **Lookback window:** 1 year (ignore older sales)
- **Minimum sales:** Require 5+ sales for confidence

**Example:**
- Sale 30 days ago: 100% weight
- Sale 90 days ago: 50% weight (half-life)
- Sale 180 days ago: 25% weight
- Sale 365 days ago: 6% weight (nearly ignored)

---

### 2. PSA-Only Search Strategy

**Requirement:** Focus on PSA graded cards only, ignore BGS/SGC/CGC

**eBay Search Keywords:**
```javascript
// CORRECT (PSA-specific)
const query = `${playerName} ${year} ${set} PSA`;

// WRONG (catches other graders)
const query = `${playerName} ${year} ${set} gem mint 10`;
```

**Filtering Logic:**
```javascript
async function searchPSACards(playerName, year, set) {
  const response = await axios.get(
    'https://api.ebay.com/buy/browse/v1/item_summary/search',
    {
      params: {
        q: `${playerName} ${year} ${set} PSA`,
        category_ids: '261328',  // Sports Trading Card Singles
        filter: 'conditions:{NEW}'  // Graded cards marked as "New"
      }
    }
  );
  
  // Additional filtering: only keep items with "PSA" in title
  return response.data.itemSummaries.filter(item => 
    item.title.toUpperCase().includes('PSA')
  );
}
```

**Database Schema Update:**
```sql
-- Add grader constraint
ALTER TABLE ebay_sales ADD CONSTRAINT check_grader_psa 
  CHECK (grader IS NULL OR grader = 'PSA');

-- Index for PSA-only queries
CREATE INDEX idx_psa_sales ON ebay_sales(card_id, sale_date) 
  WHERE grader = 'PSA' AND graded = true;
```

---

### 3. Condition Sensitivity Analysis

**Problem:** Some cards have HUGE PSA 10 premiums because raw cards are typically damaged.

**Example:**
- **High condition sensitivity:** 1952 Topps Mickey Mantle
  - PSA 10: $10M
  - PSA 9: $1M (90% discount)
  - Raw: $50K (99.5% discount)
  - **Why:** Raw copies are almost always damaged (70+ years old)

- **Low condition sensitivity:** 2023 Panini Prizm Rookie
  - PSA 10: $500
  - PSA 9: $300 (40% discount)
  - Raw: $200 (60% discount)
  - **Why:** Modern cards are often well-preserved

**Metric: PSA 10 Premium Ratio**
```sql
-- Calculate premium for PSA 10 vs PSA 9
CREATE VIEW card_condition_sensitivity AS
SELECT 
  c.id,
  c.player_name,
  c.year,
  c.set_name,
  
  -- Average PSA 10 price
  (SELECT AVG(sale_price) FROM ebay_sales 
   WHERE card_id = c.id AND grader = 'PSA' AND grade = '10' 
   AND sale_date > NOW() - INTERVAL '6 months') as avg_psa10_price,
   
  -- Average PSA 9 price
  (SELECT AVG(sale_price) FROM ebay_sales 
   WHERE card_id = c.id AND grader = 'PSA' AND grade = '9' 
   AND sale_date > NOW() - INTERVAL '6 months') as avg_psa9_price,
   
  -- Premium ratio (1.5 = 50% premium, 2.0 = 100% premium)
  (SELECT AVG(sale_price) FROM ebay_sales WHERE card_id = c.id AND grader = 'PSA' AND grade = '10') / 
  NULLIF((SELECT AVG(sale_price) FROM ebay_sales WHERE card_id = c.id AND grader = 'PSA' AND grade = '9'), 0) as psa10_premium_ratio
  
FROM cards c
WHERE EXISTS (
  SELECT 1 FROM ebay_sales 
  WHERE card_id = c.id AND grader = 'PSA' AND grade IN ('9', '10')
);
```

**Interpretation:**
- **1.0-1.3x:** Low sensitivity (modern cards, easy to grade)
- **1.3-2.0x:** Medium sensitivity (typical)
- **2.0-5.0x:** High sensitivity (vintage, condition-critical)
- **5.0x+:** Extreme sensitivity (ultra-rare, fragile)

**Use Case:** Flag opportunities differently based on sensitivity
- High sensitivity cards: Be more conservative (raw might be damaged)
- Low sensitivity cards: Be more aggressive (raw likely gradeable)

---

### 4. Filterable Opportunity Finder

**Requirement:** Toggle opportunities by sport, set, and year (not just catch-all)

**UI Concept:**
```
┌─────────────────────────────────────────────┐
│  Arbitrage Opportunity Finder               │
├─────────────────────────────────────────────┤
│                                             │
│  Sport:     [Football ▼]  [Basketball] [Baseball] [All]  │
│                                             │
│  Year:      [2023 ▼]  [2022] [2021] [All] │
│                                             │
│  Set:       [Prizm ▼]  [Optic] [Chrome] [Select] [All] │
│                                             │
│  Min Discount: [20%]                       │
│  Min PSA 10 Price: [$100]                  │
│  Condition Sensitivity: [Medium ▼]         │
│                                             │
│  [Find Opportunities]                      │
└─────────────────────────────────────────────┘

Results:
┌────────────────────────────────────────────────────────┐
│  2023 Panini Prizm Patrick Mahomes #290               │
│  PSA 10 Avg: $350 | Raw Avg: $210 | Discount: 40%    │
│  Condition Sensitivity: Low (1.4x premium)            │
│  Recent Sales: 23 (last 30 days)                      │
│  [View Details]                                        │
└────────────────────────────────────────────────────────┘
```

**SQL Query (Filterable):**
```sql
-- Find arbitrage opportunities with filters
SELECT 
  c.player_name,
  c.year,
  c.manufacturer,
  c.set_name,
  c.card_number,
  c.sport,
  
  -- PSA 10 pricing
  psa10.avg_price as psa10_avg,
  psa10.recent_sales as psa10_sales_count,
  
  -- Raw pricing
  raw.avg_price as raw_avg,
  raw.recent_sales as raw_sales_count,
  
  -- Discount percentage
  ROUND(100 * (1 - raw.avg_price / psa10.avg_price), 2) as discount_pct,
  
  -- Condition sensitivity
  sens.psa10_premium_ratio,
  CASE 
    WHEN sens.psa10_premium_ratio < 1.3 THEN 'Low'
    WHEN sens.psa10_premium_ratio < 2.0 THEN 'Medium'
    WHEN sens.psa10_premium_ratio < 5.0 THEN 'High'
    ELSE 'Extreme'
  END as condition_sensitivity,
  
  -- Time since last sale (flag stale data)
  psa10.days_since_last_sale

FROM cards c

-- PSA 10 pricing (time-weighted)
LEFT JOIN LATERAL (
  SELECT 
    SUM(sale_price * EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400))) / 
    SUM(EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400))) as avg_price,
    COUNT(*) as recent_sales,
    EXTRACT(DAY FROM NOW() - MAX(sale_date)) as days_since_last_sale
  FROM ebay_sales
  WHERE card_id = c.id
    AND grader = 'PSA'
    AND grade = '10'
    AND sale_date > NOW() - INTERVAL '6 months'
) psa10 ON true

-- Raw card pricing (time-weighted)
LEFT JOIN LATERAL (
  SELECT 
    SUM(sale_price * EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400))) / 
    SUM(EXP(-0.693 * EXTRACT(EPOCH FROM (NOW() - sale_date)) / (90 * 86400))) as avg_price,
    COUNT(*) as recent_sales
  FROM ebay_sales
  WHERE card_id = c.id
    AND graded = false
    AND sale_date > NOW() - INTERVAL '6 months'
) raw ON true

-- Condition sensitivity
LEFT JOIN card_condition_sensitivity sens ON sens.id = c.id

WHERE 
  -- Filters (dynamically added based on UI selection)
  c.sport = COALESCE($sport, c.sport)  -- e.g., 'Football'
  AND c.year = COALESCE($year, c.year)  -- e.g., '2023'
  AND c.set_name = COALESCE($set, c.set_name)  -- e.g., 'Prizm'
  
  -- Require minimum data quality
  AND psa10.recent_sales >= 5
  AND raw.recent_sales >= 3
  AND psa10.avg_price >= $min_psa10_price  -- e.g., 100.00
  
  -- Require minimum discount
  AND (1 - raw.avg_price / psa10.avg_price) >= $min_discount  -- e.g., 0.20 (20%)
  
  -- Filter by condition sensitivity
  AND sens.psa10_premium_ratio >= $min_sensitivity  -- e.g., 1.0
  AND sens.psa10_premium_ratio <= $max_sensitivity  -- e.g., 3.0

ORDER BY 
  (1 - raw.avg_price / psa10.avg_price) DESC  -- Largest discounts first
LIMIT 100;
```

**Filter Examples:**

**Scenario 1: Modern Football (Low Risk)**
```javascript
{
  sport: 'Football',
  year: '2023',
  set: 'Prizm',
  min_discount: 0.20,  // 20%+
  min_psa10_price: 100,
  min_sensitivity: 1.0,
  max_sensitivity: 1.5  // Low sensitivity only
}
```

**Scenario 2: Vintage Baseball (High Risk, High Reward)**
```javascript
{
  sport: 'Baseball',
  year: null,  // All years
  set: null,   // All sets
  min_discount: 0.50,  // 50%+ discount
  min_psa10_price: 1000,  // High-value only
  min_sensitivity: 3.0,   // High sensitivity
  max_sensitivity: 10.0
}
```

**Scenario 3: Basketball Rookies (Balanced)**
```javascript
{
  sport: 'Basketball',
  year: '2023',
  set: null,  // All sets
  min_discount: 0.30,  // 30%+
  min_psa10_price: 200,
  min_sensitivity: 1.3,
  max_sensitivity: 2.5  // Medium sensitivity
}
```

---

## REVISED DATABASE SCHEMA

### Updated Tables

```sql
-- Cards table (add sport/set/year indexes)
CREATE TABLE cards (
  id SERIAL PRIMARY KEY,
  player_name VARCHAR(200) NOT NULL,
  year VARCHAR(10),
  manufacturer VARCHAR(100),
  set_name VARCHAR(200),
  card_number VARCHAR(50),
  sport VARCHAR(50),  -- Football, Basketball, Baseball, Hockey, etc.
  rookie_card BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  
  -- Indexes for filtering
  INDEX idx_sport (sport),
  INDEX idx_year (year),
  INDEX idx_set (set_name),
  INDEX idx_sport_year_set (sport, year, set_name)
);

-- Sales table (PSA-focused)
CREATE TABLE ebay_sales (
  id SERIAL PRIMARY KEY,
  card_id INT REFERENCES cards(id),
  item_id VARCHAR(100) UNIQUE,
  title TEXT,
  sale_price DECIMAL(10,2),
  sale_date TIMESTAMP,
  graded BOOLEAN,
  grade VARCHAR(10),
  grader VARCHAR(50) CHECK (grader IS NULL OR grader = 'PSA'),  -- PSA only
  source VARCHAR(50) DEFAULT 'ebay',
  url TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  
  -- Indexes for queries
  INDEX idx_card_sales (card_id, sale_date),
  INDEX idx_psa_sales (card_id, graded, grader, grade) WHERE grader = 'PSA'
);

-- Condition sensitivity view
CREATE MATERIALIZED VIEW card_condition_sensitivity AS
SELECT 
  c.id,
  c.player_name,
  c.year,
  c.set_name,
  c.sport,
  
  -- PSA 10 vs PSA 9 premium
  (SELECT AVG(sale_price) FROM ebay_sales WHERE card_id = c.id AND grader = 'PSA' AND grade = '10' AND sale_date > NOW() - INTERVAL '6 months') as avg_psa10_price,
  (SELECT AVG(sale_price) FROM ebay_sales WHERE card_id = c.id AND grader = 'PSA' AND grade = '9' AND sale_date > NOW() - INTERVAL '6 months') as avg_psa9_price,
  
  -- Premium ratio
  (SELECT AVG(sale_price) FROM ebay_sales WHERE card_id = c.id AND grader = 'PSA' AND grade = '10') / 
  NULLIF((SELECT AVG(sale_price) FROM ebay_sales WHERE card_id = c.id AND grader = 'PSA' AND grade = '9'), 0) as psa10_premium_ratio,
  
  -- Sample size
  (SELECT COUNT(*) FROM ebay_sales WHERE card_id = c.id AND grader = 'PSA' AND grade = '10' AND sale_date > NOW() - INTERVAL '6 months') as psa10_sample_size,
  (SELECT COUNT(*) FROM ebay_sales WHERE card_id = c.id AND grader = 'PSA' AND grade = '9' AND sale_date > NOW() - INTERVAL '6 months') as psa9_sample_size
  
FROM cards c;

-- Refresh daily
CREATE INDEX idx_sensitivity_card ON card_condition_sensitivity(id);
```

---

## IMPLEMENTATION PLAN

### Phase 1: Data Collection (Week 1)
- [ ] eBay scraper with PSA keyword filtering
- [ ] Parse sport/set/year from titles (NLP or regex)
- [ ] Collect PSA 10, PSA 9, and raw sales
- [ ] Store in PostgreSQL

### Phase 2: Pricing Algorithm (Week 2)
- [ ] Implement time-weighted pricing function
- [ ] Calculate condition sensitivity ratios
- [ ] Test on sample data (validate with real market)

### Phase 3: Opportunity Finder (Week 3)
- [ ] Build filterable query (sport/set/year)
- [ ] Create simple web UI or SQL query templates
- [ ] Add alerts (email/SMS when new opportunities found)

### Phase 4: Refinement (Week 4)
- [ ] Tune time decay parameters (90 days half-life?)
- [ ] Add more filters (player position, team, parallels)
- [ ] Backtest: Would flagged opportunities have been profitable?

---

## NEXT STEPS

**Answer these questions:**

1. **Sports focus?** (Football only, or Football + Basketball + Baseball?)
2. **Target discount threshold?** (20%? 30%? 50%?)
3. **Minimum PSA 10 price?** (Only track cards worth $100+? $500+?)
4. **Time decay half-life?** (90 days? 60 days? Varies by sport?)
5. **Condition sensitivity range?** (Focus on low sensitivity cards only? Or include high sensitivity for bigger upside?)

**Once you answer, I'll:**
- Create starter code with eBay scraper
- Build SQL functions for pricing + opportunity finder
- Set up simple query interface

---

**This approach is more realistic than the original venture-backed plan.**

Focus on PSA-only arbitrage with smart filtering = actually achievable solo project.
