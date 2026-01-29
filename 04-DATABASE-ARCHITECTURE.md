# Database Architecture & ETL Strategy

**Last Updated:** January 29, 2026  
**Scope:** Complete data architecture for sports card aggregation platform

---

## EXECUTIVE SUMMARY

A sports card data platform requires a **hybrid database architecture** combining:
- **PostgreSQL** for structured transactional data (cards, users, sales)
- **TimescaleDB** (PostgreSQL extension) for time-series pricing data
- **Redis** for real-time caching and session management
- **Elasticsearch** for fast, full-text card search
- **S3/Object Storage** for images and bulk data backups

The ETL pipeline must handle **millions of daily data points** from multiple sources (eBay, PSA, auction houses), deduplicate transactions, parse unstructured text, and update pricing algorithms in near real-time.

**Estimated Scale (Year 1):**
- 10M+ cards in database
- 100M+ historical sales records
- 50K+ daily new listings
- 10K+ active users
- 500GB-1TB data storage

---

## RECOMMENDED TECH STACK

### Primary Database: PostgreSQL 16+

**Why PostgreSQL:**
- ✅ Mature, battle-tested RDBMS
- ✅ Excellent JSON support (hybrid relational + document store)
- ✅ Strong indexing capabilities (B-tree, GiST, GIN, BRIN)
- ✅ ACID compliance for financial data (user purchases, pricing)
- ✅ Rich extension ecosystem (TimescaleDB, pgvector for AI)
- ✅ Free and open source
- ✅ Excellent performance up to billions of rows

**Alternatives Considered:**
- ❌ **MongoDB:** Great for flexibility but weak for complex queries, relationships, and aggregations needed for pricing analytics
- ❌ **MySQL:** Good but PostgreSQL has better JSON support and extension ecosystem
- ⚠️ **CockroachDB:** Excellent for distributed systems but overkill for Year 1, more expensive

**Use Cases:**
- Card catalog (players, sets, manufacturers)
- User accounts and authentication
- Collection tracking
- Sales transactions
- Relationships between entities

### Time-Series Database: TimescaleDB

**Why TimescaleDB:**
- ✅ PostgreSQL extension (familiar SQL interface)
- ✅ Optimized for time-series data (pricing history, population growth)
- ✅ Automatic partitioning by time (hypertables)
- ✅ Fast queries on recent data
- ✅ Compression for historical data (10x-20x reduction)
- ✅ Continuous aggregations (pre-computed rollups)

**Alternatives:**
- **InfluxDB:** Great for metrics but not ideal for complex queries
- **Prometheus:** For monitoring, not business data

**Use Cases:**
- Hourly/daily price snapshots
- Population report history
- Market index calculations
- User portfolio value history

### Cache Layer: Redis 7+

**Why Redis:**
- ✅ Sub-millisecond response times
- ✅ Simple key-value store for hot data
- ✅ TTL support (auto-expire cached data)
- ✅ Pub/sub for real-time updates
- ✅ Sorted sets for leaderboards (top gainers/losers)

**Use Cases:**
- Cache card details (TTL: 1 hour)
- Cache pricing data (TTL: 15 minutes)
- Session management
- Rate limiting API calls
- Real-time leaderboards

### Search Engine: Elasticsearch 8+

**Why Elasticsearch:**
- ✅ Fast full-text search (player names, card sets)
- ✅ Fuzzy matching (typo tolerance)
- ✅ Faceted search (filters by year, grade, set)
- ✅ Autocomplete/type-ahead
- ✅ Scales horizontally

**Alternatives:**
- **PostgreSQL Full-Text Search:** Good for small scale, but Elasticsearch is faster
- **Algolia:** Excellent but expensive ($$$)
- **Meilisearch:** Great open-source alternative, consider for MVP

**Use Cases:**
- Card search ("2023 Panini Prizm Patrick Mahomes PSA 10")
- Autocomplete suggestions
- Filter/facet navigation

### Object Storage: AWS S3 / Cloudflare R2

**Why Object Storage:**
- ✅ Unlimited scalability
- ✅ Cheap storage ($0.02-0.03/GB/month)
- ✅ CDN integration for fast delivery
- ✅ Durability (99.999999999%)

**Use Cases:**
- Card images (front/back)
- PSA certification images
- Bulk data exports
- Database backups

### Queue System: PostgreSQL (pg_boss) or Redis Queues

**Why:**
- ✅ Reliable job processing
- ✅ Retry failed jobs
- ✅ Priority queues

**Use Cases:**
- eBay API request queues
- PSA cert verification jobs
- Email notifications
- Data enrichment tasks

---

## DATABASE SCHEMA

### Core Tables

#### 1. Players
```sql
CREATE TABLE players (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    name_normalized VARCHAR(200) NOT NULL,  -- lowercase, no accents for matching
    sport VARCHAR(50) NOT NULL,  -- Football, Basketball, Baseball, etc.
    team VARCHAR(100),
    position VARCHAR(50),
    birth_year INT,
    hall_of_fame BOOLEAN DEFAULT FALSE,
    
    -- Metadata
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_name (name),
    INDEX idx_name_normalized (name_normalized),
    INDEX idx_sport (sport),
    UNIQUE (name_normalized, sport, birth_year)
);

-- Example data
INSERT INTO players (name, name_normalized, sport, team, position, birth_year) VALUES
('Patrick Mahomes', 'patrick mahomes', 'Football', 'Kansas City Chiefs', 'QB', 1995),
('LeBron James', 'lebron james', 'Basketball', 'Los Angeles Lakers', 'F', 1984);
```

#### 2. Manufacturers
```sql
CREATE TABLE manufacturers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,  -- Panini, Topps, Upper Deck, etc.
    founded_year INT,
    country VARCHAR(50),
    active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### 3. Card Sets/Products
```sql
CREATE TABLE card_sets (
    id SERIAL PRIMARY KEY,
    manufacturer_id INT REFERENCES manufacturers(id),
    name VARCHAR(200) NOT NULL,  -- Prizm, Chrome, Optic, etc.
    year VARCHAR(10) NOT NULL,  -- "2023", "2023-24" (for multi-year)
    sport VARCHAR(50) NOT NULL,
    full_name VARCHAR(300),  -- "2023 Panini Prizm Football"
    release_date DATE,
    
    -- Metadata
    is_flagship BOOLEAN DEFAULT FALSE,  -- Major release vs niche
    estimated_print_run INT,  -- If known
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_manufacturer (manufacturer_id),
    INDEX idx_year (year),
    INDEX idx_sport (sport),
    UNIQUE (manufacturer_id, name, year, sport)
);
```

#### 4. Cards (Master Catalog)
```sql
CREATE TABLE cards (
    id SERIAL PRIMARY KEY,
    
    -- Card Identity
    card_set_id INT REFERENCES card_sets(id),
    player_id INT REFERENCES players(id),
    card_number VARCHAR(50),  -- "290", "Base Set #1", etc.
    
    -- Card Attributes
    is_rookie BOOLEAN DEFAULT FALSE,
    is_autograph BOOLEAN DEFAULT FALSE,
    is_memorabilia BOOLEAN DEFAULT FALSE,  -- Patch, jersey, etc.
    is_serial_numbered BOOLEAN DEFAULT FALSE,
    serial_number_total INT,  -- e.g., /99, /25
    
    parallel_type VARCHAR(100),  -- Base, Silver, Gold, Red, etc.
    insert_type VARCHAR(100),  -- Insert set name if applicable
    
    -- Text search
    title TEXT,  -- Generated full title for search
    description TEXT,
    
    -- Images
    image_front_url TEXT,
    image_back_url TEXT,
    
    -- Metadata
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_card_set (card_set_id),
    INDEX idx_player (player_id),
    INDEX idx_rookie (is_rookie),
    INDEX idx_autograph (is_autograph),
    INDEX idx_serial (is_serial_numbered),
    
    -- Full-text search (for PostgreSQL FTS, supplement with Elasticsearch)
    INDEX idx_title_fts (to_tsvector('english', title))
);

-- Auto-generate title
CREATE OR REPLACE FUNCTION generate_card_title() RETURNS TRIGGER AS $$
BEGIN
    NEW.title := (
        SELECT CONCAT(
            cs.year, ' ',
            m.name, ' ',
            cs.name, ' ',
            p.name,
            CASE WHEN NEW.card_number IS NOT NULL THEN CONCAT(' #', NEW.card_number) ELSE '' END,
            CASE WHEN NEW.parallel_type IS NOT NULL THEN CONCAT(' ', NEW.parallel_type) ELSE '' END,
            CASE WHEN NEW.is_rookie THEN ' RC' ELSE '' END,
            CASE WHEN NEW.is_autograph THEN ' Auto' ELSE '' END,
            CASE WHEN NEW.serial_number_total IS NOT NULL THEN CONCAT(' /', NEW.serial_number_total) ELSE '' END
        )
        FROM card_sets cs
        JOIN manufacturers m ON cs.manufacturer_id = m.id
        JOIN players p ON p.id = NEW.player_id
        WHERE cs.id = NEW.card_set_id
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_generate_card_title
    BEFORE INSERT OR UPDATE ON cards
    FOR EACH ROW
    EXECUTE FUNCTION generate_card_title();
```

#### 5. Grading Companies
```sql
CREATE TABLE grading_companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,  -- PSA, BGS, SGC, CGC, Raw
    abbreviation VARCHAR(10) NOT NULL UNIQUE,
    website TEXT,
    active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO grading_companies (name, abbreviation, website) VALUES
('Professional Sports Authenticator', 'PSA', 'https://www.psacard.com'),
('Beckett Grading Services', 'BGS', 'https://www.beckett.com/grading'),
('Sportscard Guaranty', 'SGC', 'https://www.gosgc.com'),
('Certified Guaranty Company', 'CGC', 'https://www.cgccards.com'),
('Raw/Ungraded', 'RAW', NULL);
```

#### 6. Graded Cards
```sql
CREATE TABLE graded_cards (
    id SERIAL PRIMARY KEY,
    card_id INT REFERENCES cards(id),
    grading_company_id INT REFERENCES grading_companies(id),
    grade VARCHAR(10) NOT NULL,  -- "10", "9.5", "9", "Authentic", etc.
    
    -- Certification
    cert_number VARCHAR(50),  -- PSA/BGS/SGC cert number
    cert_verified BOOLEAN DEFAULT FALSE,
    cert_verified_at TIMESTAMP,
    
    -- Sub-grades (for BGS)
    centering_grade VARCHAR(10),
    corners_grade VARCHAR(10),
    edges_grade VARCHAR(10),
    surface_grade VARCHAR(10),
    autograph_grade VARCHAR(10),  -- For dual-graded cards
    
    -- PSA Data (from API)
    psa_label_type VARCHAR(50),
    psa_image_url TEXT,
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_card (card_id),
    INDEX idx_grading_company (grading_company_id),
    INDEX idx_grade (grade),
    INDEX idx_cert_number (cert_number),
    UNIQUE (grading_company_id, cert_number)  -- Cert numbers are unique per grader
);
```

#### 7. Marketplaces
```sql
CREATE TABLE marketplaces (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    url TEXT,
    type VARCHAR(50),  -- Auction, Marketplace, Fractional, Private
    active BOOLEAN DEFAULT TRUE,
    
    -- Data source info
    has_api BOOLEAN DEFAULT FALSE,
    api_endpoint TEXT,
    requires_scraping BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO marketplaces (name, url, type, has_api, requires_scraping) VALUES
('eBay', 'https://www.ebay.com', 'Marketplace', TRUE, FALSE),
('PWCC Marketplace', 'https://www.pwccmarketplace.com', 'Auction', FALSE, TRUE),
('Goldin Auctions', 'https://goldin.co', 'Auction', FALSE, TRUE),
('Heritage Auctions', 'https://www.ha.com', 'Auction', FALSE, TRUE),
('ComC', 'https://www.comc.com', 'Marketplace', FALSE, TRUE),
('StarStock', 'https://www.starstock.com', 'Fractional', FALSE, TRUE),
('Alt', 'https://www.alt.com', 'Fractional', FALSE, TRUE);
```

#### 8. Sales Transactions
```sql
CREATE TABLE sales (
    id BIGSERIAL PRIMARY KEY,
    
    -- Identifiers
    external_id VARCHAR(255),  -- eBay item ID, PWCC lot ID, etc.
    marketplace_id INT REFERENCES marketplaces(id),
    
    -- Card Information
    graded_card_id INT REFERENCES graded_cards(id),
    card_id INT REFERENCES cards(id),  -- For raw cards
    
    -- Sale Details
    sale_price DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    shipping_cost DECIMAL(10, 2),
    buyer_premium DECIMAL(10, 2),  -- Auction house fees
    total_price DECIMAL(10, 2),  -- sale_price + shipping + premium
    
    sale_date TIMESTAMP NOT NULL,
    sale_type VARCHAR(50),  -- Auction, Buy Now, Best Offer, Make Offer
    
    -- Listing Details
    title TEXT,
    description TEXT,
    
    -- Seller Info
    seller_username VARCHAR(255),
    seller_feedback_score INT,
    seller_feedback_percent DECIMAL(5, 2),
    
    -- Metadata
    verified BOOLEAN DEFAULT FALSE,  -- Manually verified by research team
    is_outlier BOOLEAN DEFAULT FALSE,  -- Flagged as potential fake/shill
    outlier_reason TEXT,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_marketplace (marketplace_id),
    INDEX idx_graded_card (graded_card_id),
    INDEX idx_card (card_id),
    INDEX idx_sale_date (sale_date DESC),
    INDEX idx_sale_price (sale_price),
    INDEX idx_external_id (marketplace_id, external_id),
    
    UNIQUE (marketplace_id, external_id)  -- Prevent duplicate sales
);

-- Partition by month for performance
CREATE TABLE sales_2026_01 PARTITION OF sales
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- Create partitions for each month
```

#### 9. Active Listings (Current Market)
```sql
CREATE TABLE active_listings (
    id BIGSERIAL PRIMARY KEY,
    
    external_id VARCHAR(255),
    marketplace_id INT REFERENCES marketplaces(id),
    
    graded_card_id INT REFERENCES graded_cards(id),
    card_id INT REFERENCES cards(id),
    
    -- Listing Details
    current_price DECIMAL(10, 2) NOT NULL,
    buy_now_price DECIMAL(10, 2),
    current_bid DECIMAL(10, 2),
    bid_count INT DEFAULT 0,
    
    currency VARCHAR(3) DEFAULT 'USD',
    shipping_cost DECIMAL(10, 2),
    
    listing_type VARCHAR(50),  -- Auction, Buy Now, Best Offer
    
    start_date TIMESTAMP,
    end_date TIMESTAMP,
    
    title TEXT,
    description TEXT,
    image_urls TEXT[],
    
    seller_username VARCHAR(255),
    
    -- Tracking
    first_seen_at TIMESTAMP DEFAULT NOW(),
    last_checked_at TIMESTAMP DEFAULT NOW(),
    ended BOOLEAN DEFAULT FALSE,
    sold BOOLEAN DEFAULT FALSE,
    
    INDEX idx_marketplace (marketplace_id),
    INDEX idx_graded_card (graded_card_id),
    INDEX idx_end_date (end_date),
    INDEX idx_ended (ended),
    
    UNIQUE (marketplace_id, external_id)
);

-- When listing ends, move to sales table
CREATE OR REPLACE FUNCTION archive_ended_listing() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.ended = TRUE AND NEW.sold = TRUE AND OLD.sold = FALSE THEN
        INSERT INTO sales (
            external_id, marketplace_id, graded_card_id, card_id,
            sale_price, currency, shipping_cost, sale_date, sale_type,
            title, description, seller_username
        ) VALUES (
            NEW.external_id, NEW.marketplace_id, NEW.graded_card_id, NEW.card_id,
            COALESCE(NEW.current_bid, NEW.current_price), NEW.currency, NEW.shipping_cost,
            NEW.end_date, NEW.listing_type, NEW.title, NEW.description, NEW.seller_username
        )
        ON CONFLICT (marketplace_id, external_id) DO NOTHING;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_archive_ended_listing
    AFTER UPDATE ON active_listings
    FOR EACH ROW
    EXECUTE FUNCTION archive_ended_listing();
```

#### 10. Pricing Data (Time-Series with TimescaleDB)
```sql
-- Create hypertable for time-series pricing
CREATE TABLE card_pricing (
    graded_card_id INT NOT NULL REFERENCES graded_cards(id),
    timestamp TIMESTAMP NOT NULL,
    
    -- Pricing Metrics
    current_value DECIMAL(10, 2),  -- Estimated market value
    avg_sale_price_7d DECIMAL(10, 2),
    avg_sale_price_30d DECIMAL(10, 2),
    avg_sale_price_90d DECIMAL(10, 2),
    avg_sale_price_1y DECIMAL(10, 2),
    
    median_sale_price_30d DECIMAL(10, 2),
    
    high_sale_30d DECIMAL(10, 2),
    low_sale_30d DECIMAL(10, 2),
    
    sale_count_7d INT,
    sale_count_30d INT,
    sale_count_90d INT,
    
    -- Active Listings
    active_listing_count INT,
    avg_active_listing_price DECIMAL(10, 2),
    min_active_listing_price DECIMAL(10, 2),
    
    -- Trend Indicators
    price_change_7d DECIMAL(10, 2),  -- Dollar change
    price_change_pct_7d DECIMAL(5, 2),  -- Percentage change
    price_change_30d DECIMAL(10, 2),
    price_change_pct_30d DECIMAL(5, 2),
    
    trend VARCHAR(20),  -- Up, Down, Stable
    volatility DECIMAL(5, 2),  -- Standard deviation / mean
    
    PRIMARY KEY (graded_card_id, timestamp)
);

-- Convert to TimescaleDB hypertable
SELECT create_hypertable('card_pricing', 'timestamp');

-- Create continuous aggregates for daily rollups
CREATE MATERIALIZED VIEW card_pricing_daily
WITH (timescaledb.continuous) AS
SELECT
    graded_card_id,
    time_bucket('1 day', timestamp) AS day,
    AVG(current_value) AS avg_value,
    MAX(high_sale_30d) AS high_sale,
    MIN(low_sale_30d) AS low_sale,
    AVG(sale_count_30d) AS avg_sales_per_day
FROM card_pricing
GROUP BY graded_card_id, day;

-- Compress old data (reduce storage by 10-20x)
SELECT add_compression_policy('card_pricing', INTERVAL '7 days');
```

#### 11. PSA Population Reports
```sql
CREATE TABLE psa_populations (
    graded_card_id INT NOT NULL,
    snapshot_date DATE NOT NULL,
    
    -- Population by Grade
    pop_total INT,
    pop_10 INT,
    pop_9_5 INT,
    pop_9 INT,
    pop_8_5 INT,
    pop_8 INT,
    pop_7_5 INT,
    pop_7 INT,
    pop_6 INT,
    pop_5 INT,
    pop_4 INT,
    pop_3 INT,
    pop_2 INT,
    pop_1 INT,
    pop_authentic INT,  -- Authentic/Altered
    
    -- Calculated Fields
    pop_higher INT,  -- For PSA 9, this would be pop_10 + pop_9_5
    
    PRIMARY KEY (graded_card_id, snapshot_date),
    FOREIGN KEY (graded_card_id) REFERENCES graded_cards(id)
);

-- Convert to hypertable for time-series
SELECT create_hypertable('psa_populations', 'snapshot_date');

-- Index for quick lookups
CREATE INDEX idx_psa_pop_card ON psa_populations(graded_card_id, snapshot_date DESC);
```

#### 12. Users
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    username VARCHAR(100) UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    
    -- Profile
    full_name VARCHAR(200),
    avatar_url TEXT,
    
    -- Subscription
    subscription_tier VARCHAR(20) DEFAULT 'free',  -- free, pro, enterprise
    subscription_status VARCHAR(20) DEFAULT 'active',
    subscription_start_date TIMESTAMP,
    subscription_end_date TIMESTAMP,
    stripe_customer_id VARCHAR(255),
    
    -- Metadata
    created_at TIMESTAMP DEFAULT NOW(),
    last_login_at TIMESTAMP,
    email_verified BOOLEAN DEFAULT FALSE,
    
    INDEX idx_email (email),
    INDEX idx_username (username),
    INDEX idx_subscription (subscription_tier, subscription_status)
);
```

#### 13. User Collections
```sql
CREATE TABLE user_collections (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    name VARCHAR(200) DEFAULT 'My Collection',
    description TEXT,
    is_public BOOLEAN DEFAULT FALSE,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_user (user_id)
);

CREATE TABLE user_collection_items (
    id BIGSERIAL PRIMARY KEY,
    collection_id INT NOT NULL REFERENCES user_collections(id) ON DELETE CASCADE,
    user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    graded_card_id INT REFERENCES graded_cards(id),
    card_id INT REFERENCES cards(id),
    
    -- Purchase Info
    quantity INT DEFAULT 1,
    purchase_price DECIMAL(10, 2),
    purchase_date DATE,
    purchase_source VARCHAR(255),
    
    -- Current Value (auto-updated)
    current_value DECIMAL(10, 2),
    last_value_update TIMESTAMP,
    
    -- Calculated
    gain_loss DECIMAL(10, 2) GENERATED ALWAYS AS (
        CASE WHEN purchase_price IS NOT NULL 
        THEN (current_value * quantity) - (purchase_price * quantity)
        ELSE NULL END
    ) STORED,
    
    gain_loss_pct DECIMAL(5, 2) GENERATED ALWAYS AS (
        CASE WHEN purchase_price IS NOT NULL AND purchase_price > 0
        THEN ((current_value - purchase_price) / purchase_price * 100)
        ELSE NULL END
    ) STORED,
    
    -- Organization
    folder VARCHAR(200),
    tags TEXT[],
    notes TEXT,
    
    added_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_collection (collection_id),
    INDEX idx_user (user_id),
    INDEX idx_graded_card (graded_card_id),
    INDEX idx_gain_loss (gain_loss DESC)
);

-- View for collection summary
CREATE VIEW user_collection_summary AS
SELECT
    c.id AS collection_id,
    c.user_id,
    c.name AS collection_name,
    COUNT(ci.id) AS total_cards,
    SUM(ci.quantity) AS total_quantity,
    SUM(ci.purchase_price * ci.quantity) AS total_invested,
    SUM(ci.current_value * ci.quantity) AS total_current_value,
    SUM(ci.gain_loss) AS total_gain_loss,
    CASE WHEN SUM(ci.purchase_price * ci.quantity) > 0
        THEN (SUM(ci.gain_loss) / SUM(ci.purchase_price * ci.quantity) * 100)
        ELSE 0
    END AS total_gain_loss_pct
FROM user_collections c
LEFT JOIN user_collection_items ci ON c.id = ci.collection_id
GROUP BY c.id, c.user_id, c.name;
```

---

## ETL PIPELINE ARCHITECTURE

### Data Flow Overview

```
[eBay API] ──┐
[PSA API] ───┼──> [Data Ingestion Layer] ──> [Processing Queue] ──> [Data Processing] ──> [PostgreSQL]
[Scrapers] ──┤                                      │                       │                   │
[Manual] ────┘                                      │                       │                   │
                                                    ↓                       ↓                   ↓
                                               [Redis Cache]          [Deduplication]    [TimescaleDB]
                                                                      [Enrichment]       [Pricing Updates]
                                                                      [Validation]
```

### Phase 1: Data Ingestion

**eBay Listings Ingestion:**
```javascript
// Scheduled job: Every hour
async function ingestEbayListings() {
    const topPlayers = await getTopPlayers(500);  // Top 500 tracked players
    
    for (const player of topPlayers) {
        for (const grade of ['PSA 10', 'PSA 9', 'BGS 9.5']) {
            const query = `${player.name} ${grade}`;
            
            // Fetch from eBay API
            const listings = await ebayAPI.search({
                q: query,
                category_ids: '261328',
                limit: 200
            });
            
            // Queue for processing
            for (const listing of listings) {
                await queue.add('process-listing', {
                    marketplace: 'ebay',
                    data: listing,
                    source: 'api',
                    fetched_at: new Date()
                });
            }
            
            await sleep(1000);  // Rate limiting
        }
    }
}
```

**PSA Cert Verification:**
```javascript
// Triggered when new listing contains PSA cert
async function verifPSACert(certNumber) {
    // Check cache first
    const cached = await redis.get(`psa:cert:${certNumber}`);
    if (cached) return JSON.parse(cached);
    
    // Call PSA API
    const psaData = await psaAPI.getCertification(certNumber);
    
    if (psaData.success) {
        // Cache for 7 days
        await redis.setex(`psa:cert:${certNumber}`, 604800, JSON.stringify(psaData.data));
        
        // Store in database
        await db.query(`
            INSERT INTO graded_cards (card_id, grading_company_id, grade, cert_number, cert_verified, psa_image_url)
            VALUES ($1, $2, $3, $4, TRUE, $5)
            ON CONFLICT (grading_company_id, cert_number) DO UPDATE
            SET cert_verified = TRUE, psa_image_url = EXCLUDED.psa_image_url
        `, [cardId, psaCompanyId, psaData.data.grading.cardGrade, certNumber, psaData.data.media.imageUrl]);
    }
    
    return psaData;
}
```

### Phase 2: Data Processing

**Listing Parser:**
```javascript
async function processListing(job) {
    const { marketplace, data, source } = job.data;
    
    // 1. Parse card details from title
    const parsed = parseCardDetails(data.title);
    
    // 2. Match to existing card or create new
    let card = await findOrCreateCard(parsed);
    
    // 3. Extract grading info
    const gradingInfo = extractGradingInfo(data.title);
    let gradedCard = null;
    
    if (gradingInfo) {
        // Verify PSA cert if provided
        if (gradingInfo.grader === 'PSA' && gradingInfo.certNumber) {
            await verifyPSACert(gradingInfo.certNumber);
        }
        
        gradedCard = await findOrCreateGradedCard({
            cardId: card.id,
            gradingCompany: gradingInfo.grader,
            grade: gradingInfo.grade,
            certNumber: gradingInfo.certNumber
        });
    }
    
    // 4. Store as active listing or sale
    if (data.status === 'sold') {
        await storeSale({
            marketplace,
            gradedCardId: gradedCard?.id,
            cardId: card.id,
            salePrice: data.price,
            saleDate: data.endDate,
            ...data
        });
    } else {
        await storeActiveListing({
            marketplace,
            gradedCardId: gradedCard?.id,
            cardId: card.id,
            currentPrice: data.price,
            endDate: data.endDate,
            ...data
        });
    }
    
    // 5. Trigger pricing update
    if (gradedCard) {
        await queue.add('update-pricing', { gradedCardId: gradedCard.id });
    }
}
```

**Card Detail Parser (NLP/Regex):**
```javascript
function parseCardDetails(title) {
    const details = {
        player: null,
        year: null,
        manufacturer: null,
        set: null,
        cardNumber: null,
        isRookie: false,
        isAutograph: false,
        parallelType: null
    };
    
    // Extract year (4 digits)
    const yearMatch = title.match(/\b(19|20)\d{2}\b/);
    if (yearMatch) details.year = yearMatch[0];
    
    // Extract manufacturer
    const manufacturers = ['Panini', 'Topps', 'Upper Deck', 'Bowman', 'Donruss'];
    for (const mfr of manufacturers) {
        if (title.toLowerCase().includes(mfr.toLowerCase())) {
            details.manufacturer = mfr;
            break;
        }
    }
    
    // Extract set
    const sets = ['Prizm', 'Optic', 'Chrome', 'Select', 'Mosaic', 'Donruss'];
    for (const set of sets) {
        if (title.toLowerCase().includes(set.toLowerCase())) {
            details.set = set;
            break;
        }
    }
    
    // Rookie card detection
    if (/\bRC\b|\bRookie\b/i.test(title)) {
        details.isRookie = true;
    }
    
    // Autograph detection
    if (/\bAuto\b|\bAutograph\b|\bSigned\b/i.test(title)) {
        details.isAutograph = true;
    }
    
    // Card number extraction
    const cardNumMatch = title.match(/#(\d+)/);
    if (cardNumMatch) details.cardNumber = cardNumMatch[1];
    
    // Player name extraction (hardest - use NLP library or database lookup)
    // For now, simple approach: extract known player names
    const knownPlayers = await getKnownPlayers();  // Cache this
    for (const player of knownPlayers) {
        if (title.toLowerCase().includes(player.name.toLowerCase())) {
            details.player = player.name;
            break;
        }
    }
    
    return details;
}
```

**Deduplication:**
```javascript
async function storeSale(saleData) {
    // Check if sale already exists
    const existing = await db.query(`
        SELECT id FROM sales
        WHERE marketplace_id = $1 AND external_id = $2
    `, [saleData.marketplace.id, saleData.externalId]);
    
    if (existing.rows.length > 0) {
        // Update existing
        await db.query(`
            UPDATE sales SET
                sale_price = $1,
                updated_at = NOW()
            WHERE id = $2
        `, [saleData.salePrice, existing.rows[0].id]);
    } else {
        // Insert new
        await db.query(`
            INSERT INTO sales (
                marketplace_id, external_id, graded_card_id, card_id,
                sale_price, sale_date, sale_type, title, seller_username
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        `, [
            saleData.marketplace.id,
            saleData.externalId,
            saleData.gradedCardId,
            saleData.cardId,
            saleData.salePrice,
            saleData.saleDate,
            saleData.saleType,
            saleData.title,
            saleData.sellerUsername
        ]);
    }
}
```

### Phase 3: Pricing Algorithm

**Calculate Current Value:**
```sql
-- Function to calculate current market value for a graded card
CREATE OR REPLACE FUNCTION calculate_card_value(p_graded_card_id INT)
RETURNS DECIMAL(10, 2) AS $$
DECLARE
    v_value DECIMAL(10, 2);
BEGIN
    SELECT
        CASE
            -- Priority 1: Recent sales (last 30 days) - weighted average
            WHEN COUNT(*) FILTER (WHERE sale_date > NOW() - INTERVAL '30 days') >= 3 THEN
                (
                    -- Weight recent sales more heavily
                    SUM(sale_price * (1 + EXTRACT(EPOCH FROM (NOW() - sale_date)) / -2592000))
                    FILTER (WHERE sale_date > NOW() - INTERVAL '30 days')
                ) / (
                    SUM(1 + EXTRACT(EPOCH FROM (NOW() - sale_date)) / -2592000)
                    FILTER (WHERE sale_date > NOW() - INTERVAL '30 days')
                )
            
            -- Priority 2: Last 90 days (if less than 3 sales in 30d)
            WHEN COUNT(*) FILTER (WHERE sale_date > NOW() - INTERVAL '90 days') >= 2 THEN
                AVG(sale_price) FILTER (WHERE sale_date > NOW() - INTERVAL '90 days')
            
            -- Priority 3: All-time average (if very few sales)
            WHEN COUNT(*) >= 1 THEN
                AVG(sale_price)
            
            -- Priority 4: Active listings average (no sales)
            ELSE
                (SELECT AVG(current_price) FROM active_listings WHERE graded_card_id = p_graded_card_id AND ended = FALSE)
        END INTO v_value
    FROM sales
    WHERE graded_card_id = p_graded_card_id
        AND verified = TRUE  -- Only verified sales
        AND is_outlier = FALSE  -- Exclude outliers
        AND sale_date > NOW() - INTERVAL '1 year';  -- Max 1 year lookback
    
    RETURN COALESCE(v_value, 0);
END;
$$ LANGUAGE plpgsql;
```

**Scheduled Pricing Updates:**
```javascript
// Run every hour
async function updateAllPricing() {
    // Get all graded cards with recent activity
    const cards = await db.query(`
        SELECT DISTINCT gc.id
        FROM graded_cards gc
        LEFT JOIN sales s ON s.graded_card_id = gc.id
        LEFT JOIN active_listings al ON al.graded_card_id = gc.id
        WHERE s.sale_date > NOW() - INTERVAL '7 days'
           OR al.last_checked_at > NOW() - INTERVAL '1 day'
    `);
    
    for (const card of cards.rows) {
        await queue.add('update-pricing', { gradedCardId: card.id }, {
            priority: 1  // High priority
        });
    }
}

// Worker to process pricing updates
async function updateCardPricing(job) {
    const { gradedCardId } = job.data;
    
    // Calculate current value
    const result = await db.query('SELECT calculate_card_value($1) AS value', [gradedCardId]);
    const currentValue = result.rows[0].value;
    
    // Calculate aggregates
    const stats = await db.query(`
        SELECT
            AVG(sale_price) FILTER (WHERE sale_date > NOW() - INTERVAL '7 days') AS avg_7d,
            AVG(sale_price) FILTER (WHERE sale_date > NOW() - INTERVAL '30 days') AS avg_30d,
            AVG(sale_price) FILTER (WHERE sale_date > NOW() - INTERVAL '90 days') AS avg_90d,
            PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY sale_price) FILTER (WHERE sale_date > NOW() - INTERVAL '30 days') AS median_30d,
            MAX(sale_price) FILTER (WHERE sale_date > NOW() - INTERVAL '30 days') AS high_30d,
            MIN(sale_price) FILTER (WHERE sale_date > NOW() - INTERVAL '30 days') AS low_30d,
            COUNT(*) FILTER (WHERE sale_date > NOW() - INTERVAL '7 days') AS count_7d,
            COUNT(*) FILTER (WHERE sale_date > NOW() - INTERVAL '30 days') AS count_30d,
            COUNT(*) FILTER (WHERE sale_date > NOW() - INTERVAL '90 days') AS count_90d
        FROM sales
        WHERE graded_card_id = $1
            AND verified = TRUE
            AND is_outlier = FALSE
    `, [gradedCardId]);
    
    const s = stats.rows[0];
    
    // Get previous value for trend calculation
    const prev = await db.query(`
        SELECT current_value
        FROM card_pricing
        WHERE graded_card_id = $1
        ORDER BY timestamp DESC
        LIMIT 1
    `, [gradedCardId]);
    
    const prevValue = prev.rows[0]?.current_value || currentValue;
    const priceChange7d = currentValue - prevValue;
    const priceChangePct7d = prevValue > 0 ? (priceChange7d / prevValue * 100) : 0;
    
    // Determine trend
    let trend = 'Stable';
    if (Math.abs(priceChangePct7d) < 5) trend = 'Stable';
    else if (priceChangePct7d > 0) trend = 'Up';
    else trend = 'Down';
    
    // Insert pricing snapshot
    await db.query(`
        INSERT INTO card_pricing (
            graded_card_id, timestamp,
            current_value,
            avg_sale_price_7d, avg_sale_price_30d, avg_sale_price_90d,
            median_sale_price_30d,
            high_sale_30d, low_sale_30d,
            sale_count_7d, sale_count_30d, sale_count_90d,
            price_change_7d, price_change_pct_7d,
            trend
        ) VALUES ($1, NOW(), $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14)
    `, [
        gradedCardId, currentValue,
        s.avg_7d, s.avg_30d, s.avg_90d,
        s.median_30d,
        s.high_30d, s.low_30d,
        s.count_7d, s.count_30d, s.count_90d,
        priceChange7d, priceChangePct7d,
        trend
    ]);
    
    // Update user collections
    await db.query(`
        UPDATE user_collection_items
        SET current_value = $1,
            last_value_update = NOW()
        WHERE graded_card_id = $2
    `, [currentValue, gradedCardId]);
    
    // Invalidate cache
    await redis.del(`card:price:${gradedCardId}`);
}
```

---

## SCALING CONSIDERATIONS

### Year 1: Small Scale (0-50K users)

**Infrastructure:**
- Single PostgreSQL instance (16GB RAM, 4 vCPUs)
- Redis (4GB RAM)
- Elasticsearch single node (8GB RAM)
- Application servers: 2-3 instances

**Costs:** ~$500-1000/month

### Year 2-3: Medium Scale (50K-500K users)

**Infrastructure:**
- PostgreSQL primary + read replicas (2-3 replicas)
- Redis cluster (3 nodes)
- Elasticsearch cluster (3 nodes)
- Application servers: Auto-scaling group (5-20 instances)
- CDN for static assets

**Optimizations:**
- Partition sales table by month
- Materialize common query results
- Aggressive caching (99%+ cache hit rate for pricing)

**Costs:** ~$3K-10K/month

### Year 3+: Large Scale (500K+ users)

**Infrastructure:**
- Multi-region PostgreSQL (primary + replicas in multiple regions)
- Sharding by sport or date ranges
- Dedicated TimescaleDB cluster
- Elasticsearch cluster (10+ nodes)
- Redis cluster (6+ nodes)
- Kubernetes for application layer

**Costs:** ~$20K-50K+/month

---

## BACKUP & DISASTER RECOVERY

**Backup Strategy:**
```sql
-- Daily full backups
pg_dump -Fc sports_cards > backup_$(date +%Y%m%d).dump

-- Continuous WAL archiving for point-in-time recovery
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://backups/wal/%f'

-- Retention: 30 days of daily backups, 12 months of monthly backups
```

**Disaster Recovery:**
- RPO (Recovery Point Objective): <1 hour
- RTO (Recovery Time Objective): <4 hours
- Multi-region replication for critical data

---

## MONITORING & OBSERVABILITY

**Key Metrics:**
- Database query performance (p95, p99 latency)
- Cache hit rate (target: >95%)
- ETL job success rate (target: >99%)
- API rate limit usage
- User collection value calculation time
- Search query latency

**Tools:**
- Datadog or New Relic for APM
- PostgreSQL pg_stat_statements for slow query analysis
- Grafana dashboards for real-time metrics

---

## SOURCES & REFERENCES

1. **PostgreSQL Documentation:** https://www.postgresql.org/docs/
2. **TimescaleDB Documentation:** https://docs.timescale.com/
3. **Database Design Patterns:** https://www.postgresql.org/docs/current/ddl.html
4. **Scaling PostgreSQL:** https://wiki.postgresql.org/wiki/Performance_Optimization

---

**BOTTOM LINE:** Use PostgreSQL + TimescaleDB as the core database, Redis for caching, Elasticsearch for search. Design schema to handle millions of sales records with efficient time-series queries. Build robust ETL pipeline with deduplication, parsing, and pricing algorithms. Plan for horizontal scaling from day one with partitioning and read replicas.
