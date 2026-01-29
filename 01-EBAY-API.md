# eBay API - Sports Card Sales Data Integration

**Last Updated:** January 29, 2026  
**Status:** Production Ready - Requires Partner Access  
**Documentation:** https://developer.ebay.com/api-docs/buy/browse/overview.html

---

## EXECUTIVE SUMMARY

eBay is the **primary data source** for sports card pricing. The platform processes millions of sports card transactions daily, making it essential for any market data aggregation platform. eBay provides multiple APIs (Browse API, Finding API, Trading API) with **5,000 calls/day default limit** for completed listings data.

**Key Challenge:** Accessing completed/sold listing data has significant restrictions and rate limiting. Production use requires eBay Partner Network approval.

---

## AUTHENTICATION

### OAuth 2.0 - Client Credentials Grant

**Endpoint:** `https://api.ebay.com/identity/v1/oauth2/token`

**Required Headers:**
- `Content-Type: application/x-www-form-urlencoded`
- `Authorization: Basic <Base64(client_id:client_secret)>`

**Request Body:**
```
grant_type=client_credentials&scope=https://api.ebay.com/oauth/api_scope/buy.item.bulk
```

**Code Example (Node.js):**
```javascript
const axios = require('axios');

async function getEbayToken(clientId, clientSecret) {
    const auth = Buffer.from(`${clientId}:${clientSecret}`).toString('base64');
    
    const response = await axios.post(
        'https://api.ebay.com/identity/v1/oauth2/token',
        'grant_type=client_credentials&scope=https://api.ebay.com/oauth/api_scope/buy.item.bulk',
        {
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
                'Authorization': `Basic ${auth}`
            }
        }
    );
    
    return response.data.access_token;
}
```

**Response:**
```json
{
    "access_token": "v^1.1#i^1#...",
    "expires_in": 7200,
    "token_type": "Bearer"
}
```

**Token expires in 2 hours** - implement refresh logic.

---

## PRIMARY APIS FOR SPORTS CARDS

### 1. Browse API (RECOMMENDED)
**Base URL:** `https://api.ebay.com/buy/browse/v1/`

**Best for:** Real-time active listings, item details, search

**Key Endpoints:**
- `GET /item_summary/search` - Search for items by keyword, category
- `GET /item/{itemId}` - Get detailed item information
- `GET /item/get_items_by_item_group` - Get all variations of an item

**Rate Limit:** 5,000 calls/day (default)

**Sports Card Specific Search Example:**
```bash
curl -X GET 'https://api.ebay.com/buy/browse/v1/item_summary/search?q=2023+Panini+Prizm+Patrick+Mahomes+PSA+10&category_ids=261328&filter=buyingOptions:{FIXED_PRICE|AUCTION},conditions:{NEW},price:[0..1000],priceCurrency:USD' \
-H 'Authorization: Bearer <access_token>' \
-H 'X-EBAY-C-MARKETPLACE-ID: EBAY_US'
```

**Important Category IDs for Sports Cards:**
- `261328` - Sports Trading Card Singles
- `183050` - Non-Sport Trading Card Singles
- `183454` - CCG Individual Cards

**Key Response Fields:**
```json
{
    "itemSummaries": [{
        "itemId": "v1|334567890123|0",
        "title": "2023 Panini Prizm Patrick Mahomes PSA 10",
        "price": {
            "value": "299.99",
            "currency": "USD"
        },
        "condition": "New",
        "itemEndDate": "2026-02-15T20:30:00.000Z",
        "seller": {
            "username": "sportscards_pro",
            "feedbackPercentage": "99.8"
        },
        "categoryPath": "Sports Mem, Cards & Fan Shop|Sports Trading Cards|Trading Card Singles",
        "image": {
            "imageUrl": "https://..."
        }
    }]
}
```

### 2. Finding API (Completed Listings)
**Base URL:** `https://svcs.ebay.com/services/search/FindingService/v1`

**Best for:** Historical completed/sold listings

**Key Method:** `findCompletedItems`

**CRITICAL LIMITATION:** This API has **aggressive rate limiting** in production. Community reports show rate-limit errors even on the first call. Consider this unreliable for high-volume use.

**Request Example (XML):**
```xml
POST https://svcs.ebay.com/services/search/FindingService/v1
Content-Type: text/xml

<?xml version="1.0" encoding="UTF-8"?>
<findCompletedItemsRequest xmlns="http://www.ebay.com/marketplace/search/v1/services">
    <keywords>2023 Panini Prizm Patrick Mahomes PSA 10</keywords>
    <categoryId>261328</categoryId>
    <itemFilter>
        <name>SoldItemsOnly</name>
        <value>true</value>
    </itemFilter>
    <itemFilter>
        <name>EndTimeFrom</name>
        <value>2026-01-01T00:00:00.000Z</value>
    </itemFilter>
    <itemFilter>
        <name>EndTimeTo</name>
        <value>2026-01-29T23:59:59.000Z</value>
    </itemFilter>
    <paginationInput>
        <entriesPerPage>100</entriesPerPage>
        <pageNumber>1</pageNumber>
    </paginationInput>
</findCompletedItemsRequest>
```

**Response includes:**
- Final sale price
- Sale date
- Shipping cost
- Bid count
- Listing format (Auction/BuyItNow)

**WORKAROUND:** Since Finding API is unreliable, consider:
1. Using eBay's **Feed API** for bulk data downloads
2. Scraping with proper rate limiting (against ToS, use cautiously)
3. Partnering with services that have negotiated access (e.g., Terapeak)

### 3. Trading API
**Best for:** Seller account management (if you need to create listings)

**Completed listings:** `GetItem` method works but has 90-day limitation - items older than 90 days don't return full data.

---

## FILTERING FOR SPORTS CARDS

### Essential Filters

**1. Category Filter:**
```
category_ids=261328  // Sports Trading Card Singles
```

**2. Condition Codes for Graded Cards:**
- `2750` - LIKE_NEW (Graded cards)
- `4000` - USED_VERY_GOOD (Ungraded cards)

**3. Keyword Strategy:**
Include in search:
- Player name
- Year
- Set name (Panini, Topps, etc.)
- Grading company (PSA, BGS, SGC)
- Grade number (PSA 10, BGS 9.5)
- Card type (Rookie, Auto, Patch)

**Example:**
```
q=2023+Panini+Prizm+Justin+Herbert+Rookie+Auto+PSA+10
```

**4. Price Range:**
```
filter=price:[10..5000],priceCurrency:USD
```

**5. Item Aspects (Advanced Filtering):**
```
aspect_filter=categoryId:261328,Player:{Patrick Mahomes},Grade:{10},Graded:{Yes}
```

---

## RATE LIMITS & COSTS

### Default Limits (Free Tier)
| API | Daily Limit | Notes |
|-----|-------------|-------|
| Browse API | 5,000 calls | Per application per day |
| Finding API | 5,000 calls | Unreliable in production |
| Trading API | 5,000 calls | Per application per day |
| Feed API | 10,000-75,000 calls | Depends on resource |

### Increasing Limits
**Application Growth Check** - Required to increase limits
- Verify API License Agreement compliance
- Prove efficient API usage
- Demonstrate business need

**Higher tiers available** after approval (10K-100K+ calls/day)

### Production Access Requirements
1. **eBay Developers Program** membership (free)
2. **Production keyset** (requires business verification)
3. **Buy API License** - Additional approval needed for Browse/Finding APIs
4. **eBay Partner Network** membership (for affiliate tracking)

### Costs
- **API Access:** FREE (within rate limits)
- **eBay Partner Network:** FREE to join
- **Higher rate limits:** Negotiated on case-by-case basis
- **No per-call fees** - only daily limits

---

## DATA STRUCTURE FOR SPORTS CARDS

### Key Fields to Capture

```json
{
    "item_id": "v1|334567890123|0",
    "legacy_item_id": "334567890123",
    "title": "2023 Panini Prizm Patrick Mahomes PSA 10 Gem Mint",
    "category_id": "261328",
    "category_path": "Sports Mem, Cards & Fan Shop|Sports Trading Cards|Trading Card Singles",
    
    "price": {
        "value": 299.99,
        "currency": "USD"
    },
    "sold_price": 285.00,  // If sold
    "shipping_cost": 4.99,
    
    "condition": "New",
    "condition_id": "2750",  // Graded card
    
    "item_specifics": {
        "player": "Patrick Mahomes",
        "team": "Kansas City Chiefs",
        "year": "2023",
        "manufacturer": "Panini",
        "set": "Prizm",
        "card_number": "290",
        "graded": "Yes",
        "grade": "10",
        "professional_grader": "PSA",
        "card_attributes": ["Rookie", "Auto", "Patch"],
        "sport": "Football"
    },
    
    "listing_info": {
        "start_date": "2026-01-15T10:00:00Z",
        "end_date": "2026-01-22T20:30:00Z",
        "listing_type": "FixedPrice",  // or "Auction"
        "buy_it_now": true,
        "best_offer_enabled": true,
        "quantity_available": 1
    },
    
    "seller": {
        "username": "sportscards_pro",
        "feedback_score": 12543,
        "positive_feedback_percent": 99.8,
        "top_rated_seller": true
    },
    
    "images": [
        "https://i.ebayimg.com/images/g/xxx/s-l1600.jpg"
    ],
    
    "psa_cert_number": "12345678",  // Extract from title/description
    "population_data": null,  // To be enriched from PSA API
    
    "fetched_at": "2026-01-29T20:00:00Z"
}
```

---

## ETL PIPELINE STRATEGY

### Phase 1: Daily Snapshot Collection

**Objective:** Capture active listings daily to build historical price database

```javascript
// Pseudo-code for daily collection
async function dailySportsCardSnapshot() {
    const categories = ['261328']; // Sports Trading Card Singles
    const topPlayers = loadTopPlayersList(); // Top 500 players to track
    
    for (const player of topPlayers) {
        for (const grade of ['PSA 10', 'PSA 9', 'BGS 9.5']) {
            const query = `${player} ${grade}`;
            
            // Search active listings
            const results = await ebayBrowseAPI.search({
                q: query,
                category_ids: categories,
                limit: 200,
                filter: 'buyingOptions:{FIXED_PRICE|AUCTION}'
            });
            
            // Store in database
            await storeListings(results, {
                snapshot_date: new Date(),
                search_query: query
            });
            
            await sleep(1000); // Rate limiting: ~5000/day = ~3.5/min
        }
    }
}
```

**Daily volume estimate:**
- 500 players × 3 grades × 1 API call = 1,500 calls/day
- Well within 5,000/day limit

### Phase 2: Completed Sales Tracking

**Challenge:** Finding API unreliable

**Solution 1 - Feed API (RECOMMENDED):**
eBay's Feed API provides **bulk downloads** of item data:

```javascript
// Request a feed file for trading cards
const feedRequest = await ebay.feed.createTask({
    feedType: 'item',
    categoryId: '261328',
    schemaVersion: '1.0',
    marketplace: 'EBAY_US'
});

// Download and process the feed file (TSV/JSON)
const feedData = await downloadFeed(feedRequest.taskId);
// Process 100K+ items in bulk
```

**Advantages:**
- Bulk data access
- Higher rate limits (75K/day for item_snapshot)
- More reliable than Finding API

**Solution 2 - Third-Party Data:**
- **Terapeak Research** (eBay's own analytics tool)
- **Worthpoint** (aggregates auction data)
- **Ximilar Trading Card API** (AI-powered pricing)

### Phase 3: Real-Time Price Monitoring

Track specific high-value cards in real-time:

```javascript
async function monitorHighValueCards() {
    const watchlist = getHighValueCardsList(); // Cards >$1000
    
    setInterval(async () => {
        for (const card of watchlist) {
            const currentListings = await ebayBrowseAPI.search({
                q: card.searchQuery,
                category_ids: '261328',
                filter: `price:[${card.min_price}..${card.max_price}]`
            });
            
            // Detect price changes
            await updatePriceHistory(card.id, currentListings);
        }
    }, 3600000); // Every hour
}
```

---

## LIMITATIONS & CHALLENGES

### Critical Issues

1. **Completed Listings Data Access**
   - Finding API has aggressive rate limiting (often unusable)
   - Browse API only shows active listings
   - 90-day data retention limit on some endpoints
   - **Impact:** Historical sales data difficult to obtain in real-time

2. **Rate Limits**
   - 5,000 calls/day is restrictive for comprehensive tracking
   - Need to prioritize high-value/popular cards
   - **Mitigation:** Use Feed API for bulk downloads

3. **Production Access Barriers**
   - Requires business verification
   - Buy API requires additional licensing
   - eBay Partner Network approval process
   - **Timeline:** 2-4 weeks for full approval

4. **Data Quality Issues**
   - Titles are unstructured (need NLP to parse)
   - Item specifics not always complete
   - No standardized grading company fields
   - **Solution:** Build parsers + manual verification

5. **Search Limitations**
   - No wildcard searches allowed
   - Maximum 10,000 items per result set
   - Only fixed-price items by default (auctions excluded after first bid)

### Best Practices

1. **Efficient Search Strategy**
   - Target specific players/sets, not broad searches
   - Use category filters always
   - Batch related searches
   - Cache results to minimize API calls

2. **Data Enrichment**
   - Extract PSA/BGS cert numbers from titles
   - Parse player names, years, sets from unstructured titles
   - Cross-reference with PSA API for population data
   - Use image recognition for card identification

3. **Error Handling**
   - Implement exponential backoff for rate limits
   - Log failed requests for retry
   - Monitor daily quota usage
   - Graceful degradation when limits hit

---

## TERMS OF SERVICE HIGHLIGHTS

**eBay API License Agreement - Key Points:**

1. **Allowed Use:**
   - Building applications that enhance eBay marketplace
   - Affiliate marketing (with Partner Network approval)
   - Research and analytics (within rate limits)

2. **Prohibited:**
   - Scraping eBay.com directly (must use APIs)
   - Circumventing rate limits
   - Reselling eBay data to third parties
   - Creating competing marketplaces

3. **Attribution:**
   - Must display "Powered by eBay" badge
   - Link back to eBay listings
   - Use affiliate tracking URLs (if monetizing)

4. **Data Retention:**
   - Can cache data for 90 days
   - Must refresh regularly
   - Cannot create permanent historical archives without permission

**Source:** https://developer.ebay.com/join/api-license-agreement

---

## RECOMMENDED APPROACH

### For Initial MVP (Months 1-3):

1. **Focus on Top 100 Players**
   - Manually curated list
   - Track PSA 9/10 and BGS 9.5/10 only
   - Minimize API calls, maximize value

2. **Use Browse API + Feed API**
   - Browse for real-time active listings (1-2K calls/day)
   - Feed API for weekly bulk downloads (catch completed sales)
   - Avoid Finding API entirely

3. **Supplement with Manual Data Collection**
   - eBay search alerts for high-value sales
   - Community reporting
   - Manual entry of notable sales

4. **Apply for Production Access Immediately**
   - Process takes 2-4 weeks
   - Start with sandbox testing
   - Document your use case thoroughly

### For Scale (Months 4-12):

1. **Negotiate Higher Rate Limits**
   - Target 25K-50K calls/day
   - Demonstrate value and compliance
   - Show efficient API usage

2. **Build Intelligent Caching**
   - Redis for hot data (cards searched recently)
   - PostgreSQL for long-term historical data
   - 6-12 hour cache TTL for most cards

3. **Implement Priority System**
   - High-value cards: hourly updates
   - Mid-tier: daily updates
   - Low-tier: weekly updates

4. **Partner Integrations**
   - Direct feeds from auction houses
   - PSA/BGS partnerships for grading data
   - Sports API for player stats correlation

---

## CODE EXAMPLES

### Complete Search Implementation

```javascript
const axios = require('axios');

class EbaySportsCardAPI {
    constructor(clientId, clientSecret) {
        this.clientId = clientId;
        this.clientSecret = clientSecret;
        this.accessToken = null;
        this.tokenExpiry = null;
    }
    
    async authenticate() {
        if (this.accessToken && this.tokenExpiry > Date.now()) {
            return this.accessToken;
        }
        
        const auth = Buffer.from(`${this.clientId}:${this.clientSecret}`).toString('base64');
        
        const response = await axios.post(
            'https://api.ebay.com/identity/v1/oauth2/token',
            'grant_type=client_credentials&scope=https://api.ebay.com/oauth/api_scope/buy.item.bulk',
            {
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded',
                    'Authorization': `Basic ${auth}`
                }
            }
        );
        
        this.accessToken = response.data.access_token;
        this.tokenExpiry = Date.now() + (response.data.expires_in * 1000);
        
        return this.accessToken;
    }
    
    async searchSportsCards({ player, year, set, grade, grader, limit = 50 }) {
        await this.authenticate();
        
        // Build search query
        const queryParts = [];
        if (player) queryParts.push(player);
        if (year) queryParts.push(year);
        if (set) queryParts.push(set);
        if (grader && grade) queryParts.push(`${grader} ${grade}`);
        
        const q = queryParts.join(' ');
        
        const response = await axios.get(
            'https://api.ebay.com/buy/browse/v1/item_summary/search',
            {
                params: {
                    q,
                    category_ids: '261328',
                    limit,
                    filter: 'buyingOptions:{FIXED_PRICE|AUCTION},conditions:{NEW}'
                },
                headers: {
                    'Authorization': `Bearer ${this.accessToken}`,
                    'X-EBAY-C-MARKETPLACE-ID': 'EBAY_US'
                }
            }
        );
        
        return this.parseResults(response.data);
    }
    
    parseResults(data) {
        if (!data.itemSummaries) return [];
        
        return data.itemSummaries.map(item => ({
            itemId: item.itemId,
            title: item.title,
            price: parseFloat(item.price?.value || 0),
            currency: item.price?.currency || 'USD',
            condition: item.condition,
            endDate: item.itemEndDate,
            seller: {
                username: item.seller?.username,
                feedbackScore: item.seller?.feedbackScore,
                feedbackPercent: item.seller?.feedbackPercentage
            },
            imageUrl: item.image?.imageUrl,
            itemUrl: item.itemWebUrl,
            
            // Extract card details from title (simple parsing)
            parsed: this.parseCardDetails(item.title)
        }));
    }
    
    parseCardDetails(title) {
        // Simple regex-based extraction (improve with NLP)
        const details = {
            player: null,
            year: null,
            grader: null,
            grade: null,
            set: null
        };
        
        // Extract year (4 digits)
        const yearMatch = title.match(/\b(19|20)\d{2}\b/);
        if (yearMatch) details.year = yearMatch[0];
        
        // Extract PSA/BGS grade
        const gradeMatch = title.match(/(PSA|BGS|SGC)\s*(\d+\.?\d*)/i);
        if (gradeMatch) {
            details.grader = gradeMatch[1].toUpperCase();
            details.grade = gradeMatch[2];
        }
        
        // Common set names
        const sets = ['Prizm', 'Optic', 'Chrome', 'Select', 'Topps', 'Panini'];
        for (const set of sets) {
            if (title.toLowerCase().includes(set.toLowerCase())) {
                details.set = set;
                break;
            }
        }
        
        return details;
    }
}

// Usage
const api = new EbaySportsCardAPI('your_client_id', 'your_client_secret');

const results = await api.searchSportsCards({
    player: 'Patrick Mahomes',
    year: '2023',
    set: 'Prizm',
    grader: 'PSA',
    grade: '10',
    limit: 100
});

console.log(`Found ${results.length} cards`);
```

---

## SOURCES & REFERENCES

1. **eBay Developer Documentation:**
   - Browse API: https://developer.ebay.com/api-docs/buy/browse/overview.html
   - Finding API: https://developer.ebay.com/devzone/finding/callref/findCompletedItems.html
   - API Call Limits: https://developer.ebay.com/develop/get-started/api-call-limits
   - OAuth Guide: https://developer.ebay.com/api-docs/static/oauth-tokens.html

2. **Community Resources:**
   - eBay Developers Community: https://community.ebay.com/t5/eBay-APIs/ct-p/ebay-apis
   - GitHub Examples: https://github.com/search?q=ebay+api+sports+cards

3. **Third-Party Tools:**
   - Terapeak Research: https://www.ebay.com/sh/research
   - Ximilar Trading Card API: https://www.ximilar.com/blog/get-an-ai-powered-trading-card-price-checker-via-api/

---

**BOTTOM LINE:** eBay is indispensable but challenging. Browse API works well for active listings. Feed API is your best bet for bulk data. Finding API is unreliable. Plan for 2-4 week approval process and invest in smart caching/prioritization to work within rate limits.
