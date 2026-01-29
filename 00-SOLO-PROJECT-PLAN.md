# Sports Card Database - Solo Project Plan (Bootstrapped)

**Budget:** $100/month maximum  
**Team:** Solo (you)  
**Scope:** Personal database for querying sports card pricing data  
**Timeline:** 2-4 weeks to functional system

---

## REVISED PROJECT SCOPE

**What This IS:**
- Personal sports card data aggregation tool
- Query interface for YOUR use (not public-facing)
- Automated data collection from eBay + PSA
- Local/cloud database you can query via SQL or simple UI
- Track cards you're interested in
- Historical pricing analysis

**What This IS NOT:**
- Public SaaS platform like CardLadder
- Multi-user system with accounts
- Public API for other developers
- Mobile apps or polished UX
- Marketing/customer acquisition
- Venture-backed startup

---

## TECH STACK (Budget-Optimized)

### Infrastructure ($50-80/month total)

**Option A: Railway.app (Recommended)**
- PostgreSQL database: $5/month (Starter plan, 1GB)
- Node.js API/scraper: $5/month (512MB)
- Static frontend (optional): FREE on Railway or Vercel
- **Total:** $10/month base (scales as needed)

**Option B: DigitalOcean**
- $6/month Droplet (1GB RAM, 25GB SSD)
- Self-host PostgreSQL + Node.js scrapers
- **Total:** $6/month

**Option C: Local + Supabase**
- Run scrapers locally (your computer)
- Supabase free tier: 500MB database, API included
- **Total:** $0/month (upgrade to $25/month if you exceed free tier)

**Storage:**
- **Cloudflare R2:** $0.015/GB/month (way cheaper than S3)
- Store 10GB of card images: ~$0.15/month
- Or skip images entirely for now

**Recommended:** Start with **Supabase free tier** + local scrapers. Upgrade to Railway if you need more database space.

---

## SIMPLIFIED ARCHITECTURE

### Database: PostgreSQL (Core Tables Only)

**Essential Tables (Keep it Simple):**
```sql
-- Cards catalog
CREATE TABLE cards (
    id SERIAL PRIMARY KEY,
    player_name VARCHAR(200),
    year VARCHAR(10),
    manufacturer VARCHAR(100),
    set_name VARCHAR(200),
    card_number VARCHAR(50),
    rookie_card BOOLEAN,
    sport VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);

-- eBay sales (historical pricing)
CREATE TABLE ebay_sales (
    id SERIAL PRIMARY KEY,
    card_id INT REFERENCES cards(id),
    item_id VARCHAR(100) UNIQUE,
    title TEXT,
    sale_price DECIMAL(10,2),
    sale_date TIMESTAMP,
    condition VARCHAR(50),
    graded BOOLEAN,
    grade VARCHAR(10),
    grader VARCHAR(50),  -- PSA, BGS, SGC, etc.
    source VARCHAR(50) DEFAULT 'ebay',
    url TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- PSA certification data
CREATE TABLE psa_certs (
    id SERIAL PRIMARY KEY,
    cert_number VARCHAR(20) UNIQUE,
    card_id INT REFERENCES cards(id),
    grade VARCHAR(10),
    subject VARCHAR(300),
    category VARCHAR(100),
    cert_date DATE,
    image_url TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Your personal watchlist
CREATE TABLE watchlist (
    id SERIAL PRIMARY KEY,
    card_id INT REFERENCES cards(id),
    target_price DECIMAL(10,2),
    notes TEXT,
    added_at TIMESTAMP DEFAULT NOW()
);
```

**That's it.** 4 tables. No user accounts, no collections, no portfolios.

---

## DATA COLLECTION STRATEGY (Free APIs + Minimal Scraping)

### Phase 1: eBay API (Free Tier)

**Daily Limit:** 5,000 calls/day (FREE)

**Focus on:**
- Top 50-100 players you care about
- Recent sales (last 30 days)
- Graded cards only (easier to match/deduplicate)

**Cron Job Strategy:**
```bash
# Run every 6 hours (4x/day = 4,000 calls used)
0 */6 * * * node /path/to/ebay-scraper.js
```

**Per Run:**
- Search 20 players × 50 cards each = 1,000 API calls
- Store new sales in database
- Skip duplicates

**Code (Simplified):**
```javascript
// ebay-scraper.js
const axios = require('axios');
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

const WATCHED_PLAYERS = [
  'Patrick Mahomes',
  'Tom Brady',
  'LeBron James',
  'Mike Trout',
  // ... add your top 50-100 players
];

async function searchEbaySales(playerName) {
  const token = await getEbayToken();
  
  const response = await axios.get(
    'https://api.ebay.com/buy/browse/v1/item_summary/search',
    {
      headers: {
        'Authorization': `Bearer ${token}`,
        'X-EBAY-C-MARKETPLACE-ID': 'EBAY_US'
      },
      params: {
        q: playerName + ' PSA',
        category_ids: '261328',
        filter: 'conditions:{NEW}',
        limit: 50
      }
    }
  );
  
  return response.data.itemSummaries || [];
}

async function storeSales(items, playerName) {
  for (const item of items) {
    // Parse title to extract year, set, card number
    const parsed = parseCardTitle(item.title);
    
    // Check if card exists, create if not
    const cardResult = await pool.query(
      'INSERT INTO cards (player_name, year, manufacturer, set_name, sport) VALUES ($1, $2, $3, $4, $5) ON CONFLICT DO NOTHING RETURNING id',
      [playerName, parsed.year, parsed.manufacturer, parsed.set, 'Football']
    );
    
    // Store sale
    await pool.query(
      'INSERT INTO ebay_sales (card_id, item_id, title, sale_price, sale_date, graded, grade, grader, url) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9) ON CONFLICT (item_id) DO NOTHING',
      [cardResult.rows[0].id, item.itemId, item.title, item.price.value, new Date(), true, parsed.grade, parsed.grader, item.itemWebUrl]
    );
  }
}

async function main() {
  console.log('Starting eBay scraper...');
  
  for (const player of WATCHED_PLAYERS) {
    console.log(`Searching ${player}...`);
    const items = await searchEbaySales(player);
    await storeSales(items, player);
    
    // Rate limiting: wait 1 second between requests
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  console.log('Scraper complete.');
}

main();
```

### Phase 2: PSA Cert Verification (Free Tier)

**Daily Limit:** 100 calls/day (FREE)

**Strategy:**
- Extract PSA cert numbers from eBay titles
- Verify cert + get official data
- Run once/day on new certs

**Code:**
```javascript
// psa-verifier.js
async function verifyCerts() {
  // Get unchecked certs from eBay sales
  const result = await pool.query(
    "SELECT DISTINCT regexp_matches(title, 'PSA ([0-9]{8,10})', 'i') as cert FROM ebay_sales WHERE grader = 'PSA' AND cert_number IS NULL LIMIT 100"
  );
  
  for (const row of result.rows) {
    const certNumber = row.cert[0];
    
    const response = await axios.get(
      `https://api.psacard.com/publicapi/cert/GetByCertNumber/${certNumber}`,
      {
        headers: {
          'Authorization': `bearer ${process.env.PSA_API_KEY}`
        }
      }
    );
    
    if (response.data.PSACert) {
      const cert = response.data.PSACert;
      
      await pool.query(
        'INSERT INTO psa_certs (cert_number, grade, subject, category, cert_date, image_url) VALUES ($1, $2, $3, $4, $5, $6) ON CONFLICT (cert_number) DO NOTHING',
        [certNumber, cert.CardGrade, cert.Subject, cert.CategoryName, cert.SpecDate, cert.FrontImageURL]
      );
    }
    
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
}
```

### Phase 3: PWCC Scraping (Optional)

**If you want PWCC data:**
- Use Puppeteer to scrape weekly marketplace listings
- Run once/week (Sunday nights after auctions close)
- Store in same `ebay_sales` table with `source='pwcc'`

**Cost:** $0 (just your time to write scraper)

---

## QUERY INTERFACE (Choose Your Style)

### Option 1: SQL Directly (Free, Instant)
Use **pgAdmin**, **TablePlus**, or **Supabase UI** to query your database.

**Example Queries:**
```sql
-- Average PSA 10 price for a card (last 30 days)
SELECT 
  c.player_name,
  c.year,
  c.set_name,
  AVG(s.sale_price) as avg_price,
  COUNT(*) as sales_count
FROM cards c
JOIN ebay_sales s ON s.card_id = c.id
WHERE s.graded = true 
  AND s.grade = '10'
  AND s.sale_date > NOW() - INTERVAL '30 days'
GROUP BY c.id
ORDER BY avg_price DESC;

-- Price history for specific card
SELECT 
  sale_date::date,
  AVG(sale_price) as daily_avg_price
FROM ebay_sales
WHERE card_id = 123
  AND graded = true
  AND grade = '10'
GROUP BY sale_date::date
ORDER BY sale_date;

-- Your watchlist with current prices
SELECT 
  c.player_name,
  c.year,
  c.set_name,
  w.target_price,
  (SELECT AVG(sale_price) FROM ebay_sales WHERE card_id = c.id AND sale_date > NOW() - INTERVAL '7 days') as current_price
FROM watchlist w
JOIN cards c ON c.id = w.card_id;
```

### Option 2: Simple Web UI (1-2 days work)

**Tech:** Next.js + Vercel (FREE hosting)

**Features:**
- Search bar (player name)
- Card list with prices
- Click card → see price chart (Chart.js)
- Add to watchlist button

**Deploy:** `vercel deploy` (FREE, $0/month)

### Option 3: Google Sheets (Easiest)

Use **Google Sheets + Apps Script** to query your database:
- Run queries on button click
- Display results in spreadsheet
- Create charts/graphs
- Share with others if needed

**Cost:** $0

---

## MONTHLY COST BREAKDOWN

### Minimal Setup ($0-10/month)
- **Database:** Supabase free tier (500MB)
- **Scrapers:** Run locally on your computer
- **Query Interface:** pgAdmin or Supabase UI (free)
- **Total:** **$0/month**

### Comfortable Setup ($25-40/month)
- **Database:** Supabase Pro ($25/month, 8GB database)
- **Scrapers:** Railway ($5/month, always-on Node.js)
- **Frontend:** Vercel free tier
- **Images:** Cloudflare R2 ($0.50/month for 10GB)
- **Total:** **$30-35/month**

### Your Budget: $100/month
- **Database:** Supabase Pro ($25/month)
- **Scrapers:** Railway ($10/month, 1GB RAM for multiple scrapers)
- **Images:** Cloudflare R2 ($2/month for 50GB)
- **Buffer:** $63/month remaining for overages or upgrades
- **Total:** **$37/month + buffer**

---

## REALISTIC TIMELINE (Solo Dev)

### Week 1: Setup & eBay Integration
**Goals:**
- Set up PostgreSQL database (Supabase)
- Create 4 core tables
- Write eBay scraper script
- Run first data collection (500-1,000 sales)

**Deliverable:** Database with first batch of eBay sales data

### Week 2: PSA Integration + Deduplication
**Goals:**
- PSA cert verification script
- Card matching algorithm (deduplicate cards)
- Watchlist functionality
- Test queries

**Deliverable:** Functional database with PSA certs linked

### Week 3: Query Interface
**Goals:**
- Choose interface (SQL client, web UI, or Sheets)
- Build simple search/filter UI if doing web
- Create saved query templates
- Set up daily scraper cron jobs

**Deliverable:** You can search and query your data easily

### Week 4: Polish + Expansion
**Goals:**
- Add more players to watchlist
- Optimize scrapers (handle errors, retry logic)
- Add price alerts (email/SMS when card hits target)
- PWCC scraper (optional)

**Deliverable:** Production-ready personal database

---

## SCOPE LIMITATIONS (What You WON'T Build)

**No Public Access:**
- No user accounts
- No public website
- No API for others
- Just you querying your database

**No Complex Features:**
- No portfolio tracking (unless you want it for yourself)
- No social features
- No mobile apps
- No payments/monetization

**No Scale Concerns:**
- 50-100 players max (not millions of cards)
- 10-50K sales records (not 100M)
- Query speed is fine with basic indexes

**No Legal/Compliance:**
- Not selling data
- Not competing with CardLadder
- Personal use = far less risk

---

## NEXT STEPS (This Week)

1. **Choose Database Host:**
   - Recommend: **Supabase free tier** (start free, upgrade if needed)
   - Alternative: **Railway** ($5/month)

2. **Apply for eBay API Keys:**
   - Go to https://developer.ebay.com
   - Create account
   - Get Sandbox keys (test)
   - Apply for Production keys (2-4 weeks approval)

3. **Get PSA API Key:**
   - Go to https://www.psacard.com/publicapi
   - Request API key (free, instant)

4. **Set Up Local Dev Environment:**
   - Install Node.js
   - Install PostgreSQL client (psql or pgAdmin)
   - Clone starter repo (I can create this for you)

5. **Write First Scraper:**
   - Start with 10 players
   - Test eBay API
   - Store results in database

6. **Create Watchlist:**
   - List 20-30 cards you want to track
   - Insert into `watchlist` table

---

## QUESTIONS TO ANSWER

Before you start building:

1. **What's your primary use case?**
   - Buying undervalued cards? (need real-time alerts)
   - Selling your collection? (need market comps)
   - Just tracking the market? (historical data is fine)

2. **How many players/cards do you care about?**
   - 10-50 players: Easy, fits free tier
   - 100-200 players: Need paid database
   - 500+ players: Probably overkill for personal use

3. **What sports?**
   - Football only: Simplest
   - Football + Basketball + Baseball: 3x the data
   - All sports: Way too much

4. **Do you need images?**
   - Yes: Add Cloudflare R2 ($1-2/month)
   - No: Save money, skip images

5. **Query interface preference?**
   - SQL directly: Fastest, free
   - Web UI: 1-2 days extra work
   - Google Sheets: Easy to share

---

## SUMMARY

**Total Budget:** $0-40/month (well under your $100 limit)  
**Time Investment:** 2-4 weeks to functional system  
**Maintenance:** 1-2 hours/week (check scrapers, update watchlist)

**This is 100% doable as a solo project.**

You don't need employees, funding, or a SaaS platform. Just:
- PostgreSQL database (free or $25/month)
- Node.js scrapers (run locally or $5-10/month hosted)
- eBay + PSA APIs (both free within limits)
- Basic SQL queries or simple web UI

**Start with Supabase free tier + local scrapers = $0/month.**

Upgrade only if you need more data or want always-on scrapers.

---

**Next:** Tell me your answers to the 5 questions above and I'll create a starter repo with working code you can run today.
