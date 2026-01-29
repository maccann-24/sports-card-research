# PSA API - Grading Data & Population Reports

**Last Updated:** January 29, 2026  
**Status:** Limited Public API Available  
**Documentation:** https://www.psacard.com/publicapi/documentation

---

## EXECUTIVE SUMMARY

PSA (Professional Sports Authenticator) is the **most trusted grading company** in the sports card industry. Their API provides access to **certification verification** data, allowing you to validate graded cards and retrieve basic card information. However, the API is **extremely limited** - it does NOT provide population report data, pricing, or comprehensive grading analytics through the public API.

**Key Limitations:**
- ✅ Can verify PSA cert numbers
- ✅ Can retrieve basic card details (player, year, set, grade)
- ❌ No population report API access
- ❌ No pricing data
- ❌ No submission tracking (without separate account API)
- ❌ 100 calls/day on free tier (very restrictive)

**Strategic Importance:** Despite limited API, PSA cert verification is **essential** for validating card authenticity. Population data must be obtained through alternative methods (web scraping, partnerships, manual collection).

---

## AUTHENTICATION

### API Token System

**Get Token:**
1. Create account at https://www.psacard.com
2. Navigate to https://www.psacard.com/publicapi
3. Generate API access token (free tier: 100 calls/day)

**No OAuth** - Simple bearer token authentication.

**Request Header:**
```
Authorization: Bearer <your_access_token>
```

**Token does not expire** - but can be regenerated anytime from the PSA website.

---

## AVAILABLE API ENDPOINTS

### Base URL
```
https://api.psacard.com/publicapi
```

### Swagger Documentation
https://api.psacard.com/publicapi/swagger

### 1. Get Certificate by Cert Number

**Endpoint:**
```
GET /publicapi/cert/GetByCertNumber/{certNumber}
```

**Example Request:**
```bash
curl -X GET 'https://api.psacard.com/publicapi/cert/GetByCertNumber/12345678' \
  -H 'Authorization: Bearer <your_token>' \
  -H 'Content-Type: application/json'
```

**Example Response (Success):**
```json
{
    "PSACert": {
        "CertNumber": 12345678,
        "CardNumber": "290",
        "CardNumberData": null,
        "YearIssued": "2023",
        "Brand": "Panini",
        "Variety": "Prizm",
        "Subject": "Patrick Mahomes",
        "Category": "Football",
        "CardGrade": "10",
        "AutographGrade": null,
        "TotalPopulation": null,
        "PopulationHigher": null,
        "SpecAttr": null,
        "LabelType": "Standard",
        "CardAttributes": "Rookie Card",
        "ImageURL": "https://images.psacard.com/...",
        "IsFlagship": true
    },
    "IsValidRequest": true,
    "ServerMessage": "Request successful"
}
```

**Response Fields Explained:**

| Field | Description | Example |
|-------|-------------|---------|
| `CertNumber` | PSA certification number | 12345678 |
| `CardNumber` | Card number in set | "290" |
| `YearIssued` | Year of card production | "2023" |
| `Brand` | Card manufacturer | "Panini", "Topps" |
| `Variety` | Specific set/product | "Prizm", "Chrome" |
| `Subject` | Player/subject name | "Patrick Mahomes" |
| `Category` | Sport category | "Football", "Basketball" |
| `CardGrade` | Numeric grade | "10", "9", "8.5" |
| `AutographGrade` | Autograph grade (if applicable) | "10", null |
| `CardAttributes` | Special attributes | "Rookie Card", "Autograph" |
| `ImageURL` | Image of graded card | Full URL |
| `LabelType` | PSA label type | "Standard", "DNA" |

**CRITICAL NOTE:** `TotalPopulation` and `PopulationHigher` are returned as `null` in public API responses. These fields exist in the data structure but are NOT populated via the public API.

**Error Response (Invalid Cert):**
```json
{
    "IsValidRequest": false,
    "ServerMessage": "Invalid CertNo",
    "PSACert": null
}
```

**Error Response (Not Found):**
```json
{
    "IsValidRequest": true,
    "ServerMessage": "No data found",
    "PSACert": null
}
```

---

## RATE LIMITS & PRICING

### Free Tier
- **100 API calls per day**
- No cost
- Requires PSA account (free to create)

### Paid Tiers
**Contact PSA directly:** [[email protected]](mailto:[email protected])

Pricing not publicly listed, but based on community reports:
- **Higher call limits available** (exact pricing unknown)
- **Custom enterprise solutions** for high-volume users
- **No published pricing tiers** - must negotiate

### Rate Limit Monitoring
- API does not return rate limit headers
- Track calls manually on your end
- **Exceeding 100/day results in HTTP 429 errors**

---

## ERROR CODES

| HTTP Status | Meaning | Action |
|-------------|---------|--------|
| `200` | Success | Check `IsValidRequest` in response body |
| `204` | Empty request data | Missing cert number parameter |
| `4xx` | Invalid request path | Check endpoint URL |
| `429` | Rate limit exceeded | Wait for next day or upgrade plan |
| `500` | Server error or invalid credentials | Check token, retry with backoff |

**Important:** Even with HTTP 200, check `IsValidRequest` field:
- `"IsValidRequest": true` = Valid request
- `"IsValidRequest": false` = Invalid cert number format

---

## POPULATION REPORT DATA (NOT AVAILABLE VIA API)

### What You Need
PSA Population Reports show:
- Total number of cards graded at each grade level (PSA 1-10)
- Total population for a specific card
- Number of cards graded higher than a given grade
- Half-grade populations (e.g., PSA 8.5, 9.5)

**This data is CRITICAL** for determining card rarity and value.

### Official Source
https://www.psacard.com/pop

**Access:** Free on PSA website (requires manual search)

### API Limitations
**No public API endpoint** for population reports. The `TotalPopulation` and `PopulationHigher` fields in cert responses are always `null`.

### Workaround Strategies

#### Option 1: Web Scraping (Use with Caution)
**PSA Population Search URL Structure:**
```
https://www.psacard.com/pop/search?category={category}&year={year}&brand={brand}&variety={variety}&search={player}
```

**Example:**
```
https://www.psacard.com/pop/search?category=football&year=2023&brand=panini&variety=prizm&search=patrick+mahomes
```

**Considerations:**
- ⚠️ PSA ToS may prohibit automated scraping
- Website structure may change without notice
- Captcha/bot detection likely
- **Rate limit aggressively** (1 request every 5-10 seconds)
- Use this only for high-value/popular cards, not bulk

**Sample scraping approach (Python with Selenium):**
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
import time

def get_psa_population(year, brand, variety, player):
    driver = webdriver.Chrome()
    
    url = f"https://www.psacard.com/pop/search?category=football&year={year}&brand={brand}&variety={variety}&search={player}"
    driver.get(url)
    
    time.sleep(3)  # Wait for page load
    
    # Parse population table
    rows = driver.find_elements(By.CSS_SELECTOR, 'table.population-table tr')
    
    population_data = []
    for row in rows:
        cells = row.find_elements(By.TAG_NAME, 'td')
        if len(cells) >= 12:  # PSA 1-10 + totals
            card_data = {
                'card_number': cells[0].text,
                'description': cells[1].text,
                'psa_10': int(cells[11].text or 0),
                'psa_9': int(cells[10].text or 0),
                # ... etc
                'total': int(cells[-1].text or 0)
            }
            population_data.append(card_data)
    
    driver.quit()
    return population_data
```

**⚠️ Legal Risk:** Scraping may violate PSA's ToS. Use only for research, not commercial purposes without permission.

#### Option 2: Manual Data Collection
- Subscribe to PSA population updates
- Manual entry for high-value cards
- Community-contributed data
- **Most legally safe approach**

#### Option 3: Third-Party Data Providers
Some services aggregate PSA population data:
- **CardLadder** - Tracks population changes over time
- **CardMavin** - Historical population data
- **GemRate** - Population trends and analytics

These services likely have partnerships or manual data collection processes.

#### Option 4: PSA Partnership/Data License
**For serious platforms:**
- Contact PSA business development
- Negotiate a data licensing agreement
- **Cost:** Likely $10K-$50K+/year (speculation based on industry norms)
- **Benefits:** Official API access, population data, real-time updates

---

## CERTIFICATION VERIFICATION USE CASES

### 1. eBay Listing Enrichment

When you find an eBay listing with PSA cert number:

```javascript
async function enrichEbayListing(listing) {
    // Extract PSA cert from title/description
    const certMatch = listing.title.match(/PSA.*?(\d{8})/i);
    
    if (certMatch) {
        const certNumber = certMatch[1];
        
        // Verify with PSA API
        const psaData = await psaAPI.getCert(certNumber);
        
        if (psaData.IsValidRequest && psaData.PSACert) {
            return {
                ...listing,
                psa_verified: true,
                psa_cert: certNumber,
                psa_grade: psaData.PSACert.CardGrade,
                psa_player: psaData.PSACert.Subject,
                psa_year: psaData.PSACert.YearIssued,
                psa_set: psaData.PSACert.Variety,
                psa_image: psaData.PSACert.ImageURL,
                card_attributes: psaData.PSACert.CardAttributes
            };
        }
    }
    
    return {
        ...listing,
        psa_verified: false
    };
}
```

### 2. Fake Card Detection

**Validate cert numbers to prevent fraud:**

```javascript
async function validateCardListing(certNumber) {
    const psaData = await psaAPI.getCert(certNumber);
    
    const validationResult = {
        is_valid: false,
        concerns: []
    };
    
    if (!psaData.IsValidRequest || !psaData.PSACert) {
        validationResult.concerns.push('Invalid or non-existent PSA cert number');
        return validationResult;
    }
    
    // Cert is valid
    validationResult.is_valid = true;
    
    // Check for red flags
    if (psaData.PSACert.LabelType === 'DNA') {
        validationResult.concerns.push('DNA authentication only (not graded)');
    }
    
    return validationResult;
}
```

### 3. Collection Management

**Track users' PSA-graded cards:**

```javascript
async function addCardToCollection(userId, certNumber) {
    const psaData = await psaAPI.getCert(certNumber);
    
    if (!psaData.IsValidRequest || !psaData.PSACert) {
        throw new Error('Invalid PSA cert number');
    }
    
    const card = {
        user_id: userId,
        psa_cert: certNumber,
        player: psaData.PSACert.Subject,
        year: psaData.PSACert.YearIssued,
        brand: psaData.PSACert.Brand,
        set: psaData.PSACert.Variety,
        card_number: psaData.PSACert.CardNumber,
        grade: psaData.PSACert.CardGrade,
        autograph_grade: psaData.PSACert.AutographGrade,
        attributes: psaData.PSACert.CardAttributes,
        image_url: psaData.PSACert.ImageURL,
        added_at: new Date()
    };
    
    await db.collection('user_cards').insert(card);
    
    // Fetch current market value from eBay
    const marketValue = await getCardMarketValue(card);
    card.estimated_value = marketValue;
    
    return card;
}
```

---

## DATABASE SCHEMA FOR PSA DATA

```sql
CREATE TABLE psa_certifications (
    cert_number BIGINT PRIMARY KEY,
    card_number VARCHAR(50),
    year_issued VARCHAR(10),
    brand VARCHAR(100),
    variety VARCHAR(200),
    subject VARCHAR(200),  -- Player name
    category VARCHAR(50),  -- Sport
    card_grade VARCHAR(10),
    autograph_grade VARCHAR(10),
    label_type VARCHAR(50),
    card_attributes TEXT,
    image_url TEXT,
    is_flagship BOOLEAN,
    
    -- Population data (populated manually or via scraping)
    total_population INT,
    population_higher INT,
    psa_10_pop INT,
    psa_9_pop INT,
    psa_8_pop INT,
    -- ... other grades
    
    -- Metadata
    verified_at TIMESTAMP DEFAULT NOW(),
    last_checked TIMESTAMP,
    
    -- Indexes
    INDEX idx_subject (subject),
    INDEX idx_year_brand (year_issued, brand),
    INDEX idx_grade (card_grade),
    INDEX idx_category (category)
);

-- Track population changes over time
CREATE TABLE psa_population_history (
    id SERIAL PRIMARY KEY,
    cert_number BIGINT REFERENCES psa_certifications(cert_number),
    snapshot_date DATE,
    total_population INT,
    psa_10_pop INT,
    psa_9_pop INT,
    -- ... other grades
    
    UNIQUE(cert_number, snapshot_date),
    INDEX idx_snapshot_date (snapshot_date)
);

-- Link PSA certs to eBay listings
CREATE TABLE ebay_listings_psa (
    ebay_item_id VARCHAR(50) PRIMARY KEY,
    psa_cert_number BIGINT REFERENCES psa_certifications(cert_number),
    psa_verified BOOLEAN DEFAULT FALSE,
    verification_date TIMESTAMP,
    
    INDEX idx_cert_number (psa_cert_number)
);
```

---

## COMPLETE CODE EXAMPLE

```javascript
const axios = require('axios');

class PSACardAPI {
    constructor(apiToken) {
        this.apiToken = apiToken;
        this.baseURL = 'https://api.psacard.com/publicapi';
        this.callCount = 0;
        this.dailyLimit = 100;
    }
    
    async getCertification(certNumber) {
        if (this.callCount >= this.dailyLimit) {
            throw new Error('Daily API limit reached (100 calls)');
        }
        
        try {
            const response = await axios.get(
                `${this.baseURL}/cert/GetByCertNumber/${certNumber}`,
                {
                    headers: {
                        'Authorization': `Bearer ${this.apiToken}`,
                        'Content-Type': 'application/json'
                    },
                    timeout: 10000
                }
            );
            
            this.callCount++;
            
            if (!response.data.IsValidRequest) {
                return {
                    success: false,
                    error: response.data.ServerMessage,
                    data: null
                };
            }
            
            if (response.data.ServerMessage === 'No data found') {
                return {
                    success: false,
                    error: 'Cert number not found',
                    data: null
                };
            }
            
            return {
                success: true,
                data: this.formatCertData(response.data.PSACert)
            };
            
        } catch (error) {
            if (error.response?.status === 429) {
                throw new Error('Rate limit exceeded');
            }
            throw error;
        }
    }
    
    formatCertData(cert) {
        return {
            certNumber: cert.CertNumber,
            card: {
                player: cert.Subject,
                year: cert.YearIssued,
                brand: cert.Brand,
                set: cert.Variety,
                cardNumber: cert.CardNumber,
                category: cert.Category,
                attributes: cert.CardAttributes
            },
            grading: {
                cardGrade: cert.CardGrade,
                autographGrade: cert.AutographGrade,
                labelType: cert.LabelType
            },
            population: {
                total: cert.TotalPopulation,  // Always null in public API
                higher: cert.PopulationHigher  // Always null in public API
            },
            media: {
                imageUrl: cert.ImageURL
            },
            meta: {
                isFlagship: cert.IsFlagship
            }
        };
    }
    
    async batchGetCerts(certNumbers) {
        const results = [];
        
        for (const certNumber of certNumbers) {
            // Rate limiting: 100/day = ~1 every 15 minutes for continuous operation
            // For batch operations, limit to prevent hitting daily cap too quickly
            
            const result = await this.getCertification(certNumber);
            results.push(result);
            
            // Small delay to be polite
            await new Promise(resolve => setTimeout(resolve, 1000));
        }
        
        return results;
    }
    
    extractCertFromText(text) {
        // Extract PSA cert number from listing title/description
        const patterns = [
            /PSA\s*#?\s*(\d{8})/i,
            /cert\s*#?\s*(\d{8})/i,
            /certification\s*#?\s*(\d{8})/i,
            /\bCert:?\s*(\d{8})/i
        ];
        
        for (const pattern of patterns) {
            const match = text.match(pattern);
            if (match) {
                return match[1];
            }
        }
        
        return null;
    }
}

// Usage Example
const psaAPI = new PSACardAPI('your_api_token_here');

// Single cert lookup
const cert = await psaAPI.getCertification('12345678');
if (cert.success) {
    console.log(`Card: ${cert.data.card.player} ${cert.data.card.year}`);
    console.log(`Grade: PSA ${cert.data.grading.cardGrade}`);
    console.log(`Set: ${cert.data.card.brand} ${cert.data.card.set}`);
}

// Extract cert from eBay listing
const listingTitle = "2023 Panini Prizm Patrick Mahomes PSA 10 Cert #12345678";
const certNumber = psaAPI.extractCertFromText(listingTitle);
if (certNumber) {
    const certData = await psaAPI.getCertification(certNumber);
    // Use certData to enrich listing
}
```

---

## INTEGRATION STRATEGY

### Daily Workflow

1. **Morning: Cert Verification Queue**
   ```javascript
   // Process yesterday's new eBay listings
   const newListings = await getNewEbayListings({ date: 'yesterday' });
   
   const certsToVerify = newListings
       .map(listing => psaAPI.extractCertFromText(listing.title))
       .filter(cert => cert !== null);
   
   // Process up to 100 certs (daily limit)
   const verifiedCerts = await psaAPI.batchGetCerts(certsToVerify.slice(0, 100));
   
   // Enrich database
   await enrichListingsWithPSAData(verifiedCerts);
   ```

2. **Weekly: Population Data Collection (Manual/Scraping)**
   ```javascript
   // For top 100 high-value cards, scrape population data weekly
   const topCards = await getTopValueCards({ limit: 100 });
   
   for (const card of topCards) {
       const popData = await scrapePSAPopulation(card);  // Manual or automated
       await updatePopulationHistory(card.cert_number, popData);
       
       await sleep(10000);  // 10 second delay between scrapes
   }
   ```

3. **Real-Time: User-Submitted Certs**
   ```javascript
   // When user adds a card to collection, verify immediately
   app.post('/api/collection/add', async (req, res) => {
       const { certNumber } = req.body;
       
       const certData = await psaAPI.getCertification(certNumber);
       
       if (!certData.success) {
           return res.status(400).json({ error: 'Invalid cert number' });
       }
       
       const card = await addToUserCollection(req.user.id, certData.data);
       res.json({ card });
   });
   ```

---

## ALTERNATIVES TO PSA API

### Other Grading Companies

**BGS (Beckett Grading Services):**
- **No public API** available
- Must scrape or partner
- BGS cert lookup: https://www.beckett.com/grading/cert-verification

**SGC (Sportscard Guaranty):**
- **No public API** available
- Cert lookup: https://www.gosgc.com/cert-verification

**CGC (Certified Guaranty Company):**
- Entering sports card market
- **No API yet**

### Unified Grading Data Strategy

Since each grader has limited/no API:

1. **Build unified grading database:**
   ```sql
   CREATE TABLE graded_cards (
       id SERIAL PRIMARY KEY,
       grader VARCHAR(20),  -- PSA, BGS, SGC, CGC
       cert_number VARCHAR(50),
       grade VARCHAR(10),
       player VARCHAR(200),
       year VARCHAR(10),
       brand VARCHAR(100),
       set_name VARCHAR(200),
       card_number VARCHAR(50),
       
       UNIQUE(grader, cert_number)
   );
   ```

2. **Extract cert info from eBay titles:**
   ```javascript
   function parseGradingInfo(title) {
       const graders = {
           PSA: /PSA\s*#?\s*(\d{8})/i,
           BGS: /BGS\s*#?\s*(\d{7,10})/i,
           SGC: /SGC\s*#?\s*(\d{7,10})/i
       };
       
       for (const [grader, pattern] of Object.entries(graders)) {
           const match = title.match(pattern);
           if (match) {
               return { grader, certNumber: match[1] };
           }
       }
       
       return null;
   }
   ```

3. **Prioritize PSA verification** (only one with API), manual/scraping for others

---

## LIMITATIONS SUMMARY

| Feature | Available? | Workaround |
|---------|-----------|------------|
| Cert verification | ✅ Yes (API) | N/A |
| Card details | ✅ Yes (API) | N/A |
| Card images | ✅ Yes (API) | N/A |
| Population reports | ❌ No | Web scraping / Manual / Partnerships |
| Population history | ❌ No | Build own tracking system |
| Pricing data | ❌ No | Use eBay/market data |
| Submission tracking | ❌ No | PSA has separate authenticated API for submitters |
| Bulk cert lookup | ⚠️ Limited | Only 100/day free tier |

---

## RECOMMENDATIONS

### For MVP (Months 1-3):
1. **Use PSA API for cert verification only**
   - Validate PSA certs in eBay listings
   - Enrich listings with verified player/year/grade data
   - 100 calls/day sufficient for testing

2. **Manual population data collection**
   - Track top 50-100 cards manually
   - Weekly updates to population spreadsheet
   - Input into database

3. **Focus on PSA 10 and PSA 9**
   - Most valuable grades
   - Highest demand
   - Easier to track than all grades

### For Scale (Months 4-12):
1. **Upgrade to paid PSA API tier**
   - Contact PSA for pricing
   - Target 1,000-5,000 calls/day

2. **Build automated population tracking**
   - Careful web scraping with rate limits
   - Daily snapshots for top 500 cards
   - Historical trending

3. **Pursue PSA data partnership**
   - Official API access for population data
   - Real-time updates
   - Legal and reliable

4. **Include BGS/SGC data**
   - Manual cert verification via their websites
   - Community-contributed data
   - Aggregate all grading companies

---

## SOURCES & REFERENCES

1. **PSA Official Resources:**
   - Public API: https://www.psacard.com/publicapi
   - API Documentation: https://www.psacard.com/publicapi/documentation
   - Swagger UI: https://api.psacard.com/publicapi/swagger
   - Population Report: https://www.psacard.com/pop
   - Cert Verification: https://www.psacard.com/cert

2. **GitHub Examples:**
   - https://github.com/brad-newman/fetch-psa-api

3. **Community:**
   - r/sportscards on Reddit
   - BlowoutForums.com

---

**BOTTOM LINE:** PSA API is useful but **very limited**. Great for cert verification and basic card data. Population reports (the really valuable data) are NOT available via API - you'll need to scrape, build partnerships, or collect manually. The 100 calls/day free tier is restrictive - upgrade for production use.
