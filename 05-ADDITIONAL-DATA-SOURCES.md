# Additional Data Sources - Beyond eBay & PSA

**Last Updated:** January 29, 2026  
**Purpose:** Identify supplementary APIs and data sources to enhance platform

---

## EXECUTIVE SUMMARY

While eBay and PSA form the foundation, **additional data sources** significantly enhance value proposition:

**Must-Have (Tier 1):**
1. **PWCC Marketplace** - Major auction house, high-end sales
2. **Sports Statistics APIs** - Player performance correlation with card values
3. **BGS/SGC Grading Data** - Complete grading ecosystem coverage

**Strong Value-Add (Tier 2):**
4. **Goldin Auctions** - Record-breaking sales, market leaders
5. **Heritage Auctions** - Vintage cards, established auction house

**Nice-to-Have (Tier 3):**
6. **Fractional Platforms** (StarStock, Alt) - Emerging market data
7. **ComC, MySlabs** - Additional marketplace coverage
8. **Social Media APIs** - Sentiment analysis, trending cards

---

## 1. PWCC MARKETPLACE

### Overview
**URL:** https://www.pwccmarketplace.com  
**Now:** Owned by Fanatics (Fanatics Collect)  
**Type:** Auction + Marketplace  
**Specialization:** High-end graded cards ($1,000+ sales common)

### Why It Matters
- **Premium Market Data:** PWCC hosts some of the highest-value sports card auctions
- **Vault Integration:** Many serious collectors store cards in PWCC Vault
- **Price Leadership:** PWCC sales often set market benchmarks for rare cards
- **Volume:** Weekly auctions with thousands of lots

### Data Available

**Public Data (Website):**
- Active auctions with current bids
- Ended auctions with final sale prices
- Historical sales records
- Seller reputation scores

**Data Points:**
- Lot number
- Card details (player, year, set, grade)
- Starting bid
- Current bid / final sale price
- Number of bids
- Auction end date
- High-resolution images (front/back)
- Cert numbers (PSA/BGS/SGC)

### API Availability
**Status:** ❌ No public API

**Workaround:** Web scraping

**Scraping Strategy:**
```javascript
// PWCC auction scraping (use cautiously, respect ToS)
async function scrapePWCCAuctions() {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    
    // Navigate to weekly auction
    await page.goto('https://www.pwccmarketplace.com/weekly-auction');
    
    // Extract lot data
    const lots = await page.evaluate(() => {
        const lotElements = document.querySelectorAll('.lot-item');
        return Array.from(lotElements).map(lot => ({
            lotNumber: lot.querySelector('.lot-number')?.innerText,
            title: lot.querySelector('.lot-title')?.innerText,
            currentBid: lot.querySelector('.current-bid')?.innerText,
            bidCount: lot.querySelector('.bid-count')?.innerText,
            endTime: lot.querySelector('.end-time')?.innerText,
            imageUrl: lot.querySelector('img')?.src
        }));
    });
    
    await browser.close();
    return lots;
}
```

**Rate Limiting:**
- Scrape daily or weekly (not hourly)
- Delay 3-5 seconds between requests
- Use rotating proxies if scaled up
- Respect robots.txt

**Legal Considerations:**
- ⚠️ Scraping may violate ToS
- Consider partnership/data licensing for commercial use
- PWCC may block aggressive scrapers

### Integration Priority
**Priority:** 🔥 HIGH

**Rationale:**
- Complements eBay with high-end market data
- Serious collectors track PWCC sales
- Relatively small dataset (thousands of lots/week vs millions on eBay)
- Manageable scraping volume

**Implementation Timeline:** Month 2-3

---

## 2. SPORTS STATISTICS APIS

### Overview
**Purpose:** Correlate player performance with card values  
**Value Proposition:** Predict card price movements based on on-field performance

### Why It Matters
- **Performance Drives Value:** Player stats directly impact card demand
- **Injury/Breakout Detection:** Major events cause price spikes/crashes
- **Predictive Analytics:** ML models can forecast price changes
- **Content Creation:** Stats-driven insights for users

### Recommended APIs

#### A. ESPN API (Unofficial)
**URL:** http://site.api.espn.com/  
**Coverage:** NFL, NBA, MLB, NHL  
**Cost:** FREE (unofficial, no auth required)

**Endpoints:**
```javascript
// Example: Get NBA player stats
const url = 'http://site.api.espn.com/apis/common/v3/sports/basketball/nba/athletes/3975346';
// Returns: LeBron James career stats, recent performance, injury status

// NFL player stats
const url = 'http://site.api.espn.com/apis/common/v3/sports/football/nfl/athletes/3139477';
// Returns: Patrick Mahomes stats
```

**Data Points:**
- Player bio (team, position, height, weight)
- Career stats (points, TDs, batting avg, etc.)
- Recent game logs
- Injury reports
- Awards and accolades

**Limitations:**
- ⚠️ Unofficial API (could break without notice)
- No official support or SLA
- Rate limits unknown

#### B. SportsDataIO (Official, Paid)
**URL:** https://sportsdata.io  
**Coverage:** NFL, NBA, MLB, NHL, soccer, etc.  
**Cost:** $0-$2,000+/month (tiered pricing)

**Free Tier:**
- 1,000 API calls/month
- Delayed data (15-minute delay)

**Paid Tiers:**
- Real-time data
- Play-by-play feeds
- Projections and fantasy data
- Odds and betting data

**Best For:** Production-grade, reliable stats

#### C. The Sports DB (Free)
**URL:** https://www.thesportsdb.com/api.php  
**Cost:** FREE (with optional Patreon support)

**Coverage:**
- All major sports
- Player bios, stats, team info
- Event schedules, results

**Data Quality:** Good for basic info, not as detailed as paid APIs

#### D. Balldontlie (NBA Only, Free)
**URL:** https://www.balldontlie.io  
**Cost:** FREE

**Coverage:** NBA only  
**Data:** Player stats, game logs, team info

**Good for:** Basketball card platform

### Integration Strategy

**Phase 1: Basic Integration (Month 3)**
```javascript
// Link players to stats
CREATE TABLE player_stats (
    player_id INT REFERENCES players(id),
    stat_date DATE,
    sport VARCHAR(50),
    
    -- Generic stats (sport-agnostic)
    games_played INT,
    points_scored DECIMAL,
    
    -- Sport-specific JSON
    detailed_stats JSONB,
    
    PRIMARY KEY (player_id, stat_date)
);

// Update daily
async function updatePlayerStats() {
    const players = await getActivePlayers();
    
    for (const player of players) {
        const stats = await fetchESPNStats(player.espn_id);
        
        await db.query(`
            INSERT INTO player_stats (player_id, stat_date, games_played, points_scored, detailed_stats)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (player_id, stat_date) DO UPDATE
            SET detailed_stats = EXCLUDED.detailed_stats
        `, [player.id, new Date(), stats.games, stats.points, stats]);
    }
}
```

**Phase 2: Predictive Models (Month 6-9)**
```python
# ML model: predict card price movement based on player performance
import pandas as pd
from sklearn.ensemble import RandomForestRegressor

# Features: recent stats, team performance, playoff status, injury history
# Target: 7-day price change percentage

def train_price_prediction_model():
    # Fetch historical data
    df = pd.read_sql("""
        SELECT
            ps.points_scored,
            ps.games_played,
            cp.price_change_pct_7d AS target
        FROM player_stats ps
        JOIN players p ON ps.player_id = p.id
        JOIN cards c ON c.player_id = p.id
        JOIN graded_cards gc ON gc.card_id = c.id
        JOIN card_pricing cp ON cp.graded_card_id = gc.id
        WHERE ps.stat_date = cp.timestamp::date
    """, con=db_connection)
    
    # Train model
    X = df[['points_scored', 'games_played']]
    y = df['target']
    
    model = RandomForestRegressor()
    model.fit(X, y)
    
    return model
```

**Use Cases:**
- "This player just scored 50 points - his rookie cards are likely to spike"
- "Player injured for season - sell alert"
- "Playoff performance boost - buy recommendations"

### Integration Priority
**Priority:** 🟡 MEDIUM (Month 3-6)

**Rationale:**
- Strong differentiator vs CardLadder
- Requires ML expertise to execute well
- Start simple (show basic stats on card pages)
- Evolve to predictive features

---

## 3. BGS & SGC GRADING DATA

### Overview
**Graders:**
- **BGS (Beckett Grading Services):** Major competitor to PSA
- **SGC (Sportscard Guaranty):** Growing rapidly, especially for vintage

**Why It Matters:**
- PSA is only ~60-70% of graded card market
- BGS dominates in some segments (modern, pristine grading)
- SGC strong in vintage cards
- Multi-grader support is **table stakes** for comprehensive platform

### BGS (Beckett)

**Cert Verification:** https://www.beckett.com/grading/cert-verification  
**API:** ❌ No public API

**Data Available (Manual Lookup):**
- Cert number validation
- Card details (player, year, set)
- Overall grade + sub-grades (Centering, Corners, Edges, Surface)
- Card images

**Workaround:**
```javascript
// Scrape BGS cert verification
async function verifyBGSCert(certNumber) {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    
    await page.goto(`https://www.beckett.com/grading/card-lookup?cert=${certNumber}`);
    
    // Extract data from page
    const data = await page.evaluate(() => ({
        certNumber: document.querySelector('.cert-number')?.innerText,
        grade: document.querySelector('.overall-grade')?.innerText,
        player: document.querySelector('.player-name')?.innerText,
        // ... etc
    }));
    
    await browser.close();
    return data;
}
```

**BGS Sub-Grades (Important!):**
BGS provides 4 sub-grades (each 1-10):
- Centering
- Corners
- Edges
- Surface

A BGS 9.5 with 10 sub-grades (Black Label) is much rarer/valuable than regular 9.5.

**Database Schema Addition:**
```sql
ALTER TABLE graded_cards ADD COLUMN centering_grade VARCHAR(10);
ALTER TABLE graded_cards ADD COLUMN corners_grade VARCHAR(10);
ALTER TABLE graded_cards ADD COLUMN edges_grade VARCHAR(10);
ALTER TABLE graded_cards ADD COLUMN surface_grade VARCHAR(10);
```

### SGC (Sportscard Guaranty)

**Cert Verification:** https://www.gosgc.com/cert-verification  
**API:** ❌ No public API

**Similar to BGS:** Manual lookup only

**Growing Popularity:**
- SGC grading costs less than PSA
- Faster turnaround times
- Gaining market share in vintage cards

### Integration Priority
**Priority:** 🟡 MEDIUM-HIGH (Month 2-4)

**Rationale:**
- Essential for comprehensive coverage
- Differentiator vs competitors
- Manual/scraping only (low volume compared to eBay)
- Store BGS/SGC certs extracted from eBay titles

**Strategy:**
1. **Month 2:** Extract BGS/SGC cert numbers from eBay titles, store in database
2. **Month 3:** Manual verification for high-value cards (>$500)
3. **Month 4:** Build automated scraping for cert verification
4. **Month 6:** Population report tracking (manual initially)

---

## 4. GOLDIN AUCTIONS

### Overview
**URL:** https://goldin.co  
**Type:** Premier auction house  
**Specialization:** Record-breaking sales, ultra-high-end collectibles

**Why It Matters:**
- **Market Leaders:** Goldin sets record prices for iconic cards
- **Media Coverage:** Sales generate headlines, drive market sentiment
- **Institutional Buyers:** Attracts serious collectors and investors
- **Diversification:** Not just cards (memorabilia, game-worn items)

### Data Available
- Weekly auctions (Premier, Elite)
- Final sale prices (publicly visible)
- Card images and descriptions
- Buyer's premium information

**Example Record Sales:**
- 1952 Topps Mickey Mantle PSA 9: $12.6 million
- 2003 LeBron James Rookie Patch Auto: $5.2 million

### API Availability
**Status:** ❌ No public API

**Scraping Feasibility:** ✅ Possible but low volume

**Implementation:**
```javascript
// Weekly scraping of Goldin auctions
async function scrapeGoldinAuction() {
    // Goldin auctions are weekly, not continuous
    // Scrape once after auction closes
    
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    
    await page.goto('https://goldin.co/past_auctions');
    
    // Extract lot results
    const lots = await page.$$eval('.lot-item', items => 
        items.map(item => ({
            lotNumber: item.querySelector('.lot-number')?.innerText,
            title: item.querySelector('.title')?.innerText,
            finalPrice: item.querySelector('.final-price')?.innerText,
            // ...
        }))
    );
    
    await browser.close();
    return lots;
}
```

### Integration Priority
**Priority:** 🟢 MEDIUM (Month 4-6)

**Rationale:**
- High-value data (record sales set market benchmarks)
- Low volume (weekly auctions, manageable)
- Strong PR value ("Our data includes Goldin record sales!")
- Scraping is straightforward

---

## 5. HERITAGE AUCTIONS

### Overview
**URL:** https://www.ha.com  
**Type:** Established auction house (not sports-only)  
**Specialization:** Vintage cards, rare collectibles

**Coverage:**
- Sports cards (major category)
- Comics, coins, art, memorabilia

**Why It Matters:**
- **Vintage Leader:** Dominates pre-1980 card market
- **Credibility:** 40+ years in business, trusted
- **Comprehensive Catalogs:** Detailed lot descriptions, provenance

### Data Available
- Auction results (searchable by category)
- Price archives (past sales publicly visible)
- Detailed lot descriptions
- Condition reports

### API Availability
**Status:** ⚠️ Limited API (for partners)

**Public Access:**
- Heritage has an API but requires partnership
- Public search interface accessible
- Auction results archives free to browse

**Scraping:**
- Possible but Heritage is more protective
- Use sparingly, focus on high-value lots

### Integration Priority
**Priority:** 🟡 LOW-MEDIUM (Month 6+)

**Rationale:**
- Strong for vintage, less relevant for modern cards
- Lower volume than PWCC/Goldin in modern cards
- Manual entry for record sales may suffice initially

---

## 6. FRACTIONAL OWNERSHIP PLATFORMS

### Overview
**Platforms:**
- **StarStock:** https://www.starstock.com
- **Alt:** https://www.alt.com
- **Rally:** https://rally.io
- **Dibbs:** https://dibbs.io

**Model:** Buy/sell fractions of high-value cards (like stocks)

### Why It Matters
- **Democratization:** Lower barrier to entry for expensive cards
- **Price Discovery:** Real-time trading = continuous pricing
- **Market Sentiment:** Trade volume indicates interest
- **Emerging Market:** Growing segment, forward-looking data

### Data Available

**StarStock:**
- Current "share" price
- Historical price charts
- Trading volume
- Card details (owned by platform)

**Alt:**
- Card valuations
- Ownership fractions available
- Transaction history

### API Availability
**Status:** ❌ No public APIs (all private platforms)

**Scraping:**
- Possible for current prices
- Limited historical data

### Integration Priority
**Priority:** 🟢 LOW (Month 9-12)

**Rationale:**
- Niche market (small % of overall market)
- Data quality uncertain (platform-specific valuations)
- Nice-to-have but not essential for MVP
- Future opportunity as fractional market grows

**Use Case:**
- Show fractional ownership prices alongside traditional sales
- "This card trades at $X on StarStock"

---

## 7. SOCIAL MEDIA & SENTIMENT DATA

### Overview
**Sources:**
- **Twitter/X API:** Card collector discussions, trending players
- **Reddit API:** r/sportscards, r/baseballcards, r/basketballcards
- **YouTube Data API:** Card break videos, market commentary
- **Instagram Graph API:** Influencer posts, card showcases

### Why It Matters
- **Sentiment Analysis:** Gauge market enthusiasm
- **Trend Detection:** Identify hot players/products early
- **Community Insights:** Real collector conversations
- **Content Discovery:** Curate best hobby content

### Available APIs

#### Twitter/X API
**URL:** https://developer.twitter.com/en/docs/twitter-api  
**Cost:** 
- Free tier: 1,500 tweets/month (very limited)
- Basic: $100/month (3,000 tweets/month)
- Pro: $5,000/month (1M tweets/month)

**Use Cases:**
```javascript
// Search for mentions of specific players/cards
const query = '"Patrick Mahomes" AND (PSA OR BGS) AND (rookie OR prizm)';
const tweets = await twitterAPI.search(query, { count: 100 });

// Sentiment analysis
const sentiment = await analyzeSentiment(tweets.map(t => t.text));
// Positive sentiment correlates with price increases
```

#### Reddit API
**URL:** https://www.reddit.com/dev/api  
**Cost:** FREE

**Use Cases:**
```javascript
// Monitor card subreddits for trending cards
const posts = await reddit.getSubreddit('sportscards')
    .getHot({ limit: 100 });

// Identify frequently discussed cards
const cardMentions = extractCardMentions(posts);
```

#### YouTube Data API
**URL:** https://developers.google.com/youtube/v3  
**Cost:** FREE (10,000 quota units/day)

**Use Cases:**
- Track card breaking videos (high-value pulls)
- Identify trending sets/products
- Curate educational content

### Integration Priority
**Priority:** 🟡 LOW-MEDIUM (Month 6-12)

**Rationale:**
- Strong differentiator (sentiment analysis unique)
- Requires NLP/ML expertise
- High noise-to-signal ratio
- Better as v2 feature, not MVP

**Implementation:**
```python
# Sentiment analysis for price prediction
from transformers import pipeline

sentiment_analyzer = pipeline("sentiment-analysis")

def analyze_card_sentiment(card_name):
    tweets = fetch_tweets(f'"{card_name}" card')
    sentiments = [sentiment_analyzer(tweet['text']) for tweet in tweets]
    
    positive_count = sum(1 for s in sentiments if s['label'] == 'POSITIVE')
    sentiment_score = positive_count / len(sentiments)
    
    return {
        'card': card_name,
        'sentiment_score': sentiment_score,
        'volume': len(tweets),
        'prediction': 'bullish' if sentiment_score > 0.6 else 'bearish'
    }
```

---

## 8. SUPPLEMENTARY DATA SOURCES

### A. ComC (CheckOutMyCards)
**URL:** https://www.comc.com  
**Type:** Consignment marketplace  
**Volume:** Millions of cards available

**Value:** Mid-tier marketplace, good for price discovery

**API:** ❌ No  
**Priority:** 🟢 LOW (Month 6+)

### B. MySlabs
**URL:** https://www.myslabs.com  
**Type:** Marketplace for graded cards  
**Focus:** PSA/BGS slabs

**API:** ❌ No  
**Priority:** 🟢 LOW

### C. Probstein123 (eBay Seller)
**URL:** eBay store (major consignment seller)  
**Volume:** Huge eBay presence

**Value:** Already captured via eBay API

### D. Card Shows & Private Sales
**Challenge:** No digital trail  
**Workaround:**
- User-submitted sales (honor system)
- Partner with show organizers for data
- Manual entry for major sales

---

## PRIORITIZED ROADMAP

### Month 1-2: MVP
- ✅ eBay API (active listings + Feed API)
- ✅ PSA API (cert verification)

### Month 2-3: Essential Expansion
- 🔥 PWCC scraping (weekly auctions)
- 🔥 BGS/SGC cert extraction from eBay titles
- 🔥 Basic player stats (ESPN unofficial API)

### Month 4-6: Enhanced Coverage
- 🟡 Goldin scraping (record sales)
- 🟡 BGS/SGC cert verification (automated scraping)
- 🟡 Player stats integration (paid API if needed)
- 🟡 Heritage Auctions (high-value lots only)

### Month 6-12: Advanced Features
- 🟢 Fractional platforms (StarStock, Alt)
- 🟢 Social media sentiment
- 🟢 ComC, MySlabs
- 🟢 Predictive models (stats + sentiment → price predictions)

---

## DATA QUALITY & VERIFICATION

### Challenges
1. **Scraped data unreliable** (sites change, break scrapers)
2. **Deduplication** (same sale across multiple sources)
3. **Fake sales** (shill bidding, inflated prices)
4. **Inconsistent formats** (each source structures data differently)

### Solutions

#### 1. Manual Verification Team
- Hire 2-3 researchers to verify high-value sales (>$1,000)
- Flag outliers for manual review
- Build reputation for data quality

#### 2. Deduplication Algorithm
```javascript
async function deduplicateSale(newSale) {
    // Check if sale exists across sources
    const existing = await db.query(`
        SELECT * FROM sales
        WHERE ABS(sale_price - $1) < 5  -- Within $5
          AND sale_date BETWEEN $2 - INTERVAL '2 days' AND $2 + INTERVAL '2 days'
          AND graded_card_id = $3
    `, [newSale.price, newSale.date, newSale.gradedCardId]);
    
    if (existing.rows.length > 0) {
        // Likely duplicate - prefer verified source
        if (newSale.source === 'ebay_api') {
            // Update existing with eBay data (most reliable)
            await updateSale(existing.rows[0].id, newSale);
        }
        return { isDuplicate: true };
    }
    
    return { isDuplicate: false };
}
```

#### 3. Outlier Detection
```sql
-- Flag sales >3 standard deviations from mean
WITH stats AS (
    SELECT
        graded_card_id,
        AVG(sale_price) AS avg_price,
        STDDEV(sale_price) AS stddev_price
    FROM sales
    GROUP BY graded_card_id
)
UPDATE sales s
SET is_outlier = TRUE,
    outlier_reason = 'Price outlier (>3σ)'
FROM stats st
WHERE s.graded_card_id = st.graded_card_id
  AND ABS(s.sale_price - st.avg_price) > (3 * st.stddev_price);
```

---

## COST ANALYSIS

### Year 1 Data Costs

| Source | Type | Cost | Notes |
|--------|------|------|-------|
| eBay API | Free tier | $0 | 5K calls/day sufficient for MVP |
| PSA API | Free tier | $0 | 100 calls/day, may need upgrade ($?) |
| ESPN API | Unofficial | $0 | No official pricing |
| SportsDataIO | Paid (optional) | $0-600/year | If needed for production stats |
| Twitter API | Free/Basic | $0-1,200/year | Optional, not essential |
| PWCC/Goldin Scraping | Infrastructure | $100-300/month | Proxy services, servers |
| **Total** |  | **$100-2,000/month** | Depends on scale |

### Year 2+ Data Costs

| Source | Type | Cost | Notes |
|--------|------|------|-------|
| eBay API | Higher tier | $0 (negotiated) | Likely still free with higher limits |
| PSA API | Paid tier | $500-2,000/year | Estimated, not publicly priced |
| Stats API | Premium | $3,000-10,000/year | Real-time data, all sports |
| Data Partnerships | Licensed | $10,000-50,000/year | PWCC, Goldin, Heritage data feeds |
| **Total** |  | **$1,000-5,000/month** | At scale |

---

## LEGAL & ETHICAL CONSIDERATIONS

### Web Scraping
- ⚠️ **Check ToS** of each site before scraping
- ⚠️ **Respect robots.txt**
- ⚠️ **Rate limiting** - be a good citizen
- ⚠️ **User-Agent** - identify your scraper
- ⚠️ **Risk:** Sites can block, send cease & desist

**Recommendation:** For commercial platform, pursue **data partnerships** over long-term scraping

### Data Licensing
- Some data may be copyrighted (auction house descriptions, images)
- **Use:** Aggregated pricing data generally OK (facts not copyrightable)
- **Images:** May need permission or licenses
- **Descriptions:** Rewrite or paraphrase

### User-Generated Data
- Users can submit sales, collections
- Verify before displaying publicly
- Moderate for fake/spam data

---

## RECOMMENDATIONS

### For MVP (Months 1-3)
1. **eBay + PSA only** - nail the core before expanding
2. **Extract cert numbers** from eBay titles for BGS/SGC (store but don't verify yet)
3. **Basic player stats** from free ESPN API (show on card pages)

### For Growth (Months 4-6)
1. **Add PWCC scraping** - weekly auctions, high ROI
2. **Verify BGS/SGC certs** - build comprehensive grading database
3. **Goldin scraping** - record sales for PR and benchmarking

### For Scale (Months 6-12)
1. **Negotiate data partnerships** - official feeds from PWCC, Goldin, Heritage
2. **Paid stats API** - SportsDataIO or similar for production-grade data
3. **Social sentiment** - Twitter/Reddit for predictive features
4. **Fractional platforms** - as market matures

---

## SOURCES & REFERENCES

1. **PWCC:** https://www.pwccmarketplace.com
2. **Goldin:** https://goldin.co
3. **Heritage:** https://www.ha.com
4. **SportsDataIO:** https://sportsdata.io
5. **ESPN API (Unofficial):** http://site.api.espn.com
6. **BGS:** https://www.beckett.com/grading
7. **SGC:** https://www.gosgc.com
8. **Twitter API:** https://developer.twitter.com/en/docs/twitter-api
9. **Reddit API:** https://www.reddit.com/dev/api

---

**BOTTOM LINE:** Start with eBay + PSA, add PWCC and player stats early (Month 2-3), expand to Goldin and multi-grader support by Month 6. Scraping is necessary short-term, but pursue official partnerships for long-term reliability and legal safety. Social sentiment and fractional platforms are differentiators for v2+.
