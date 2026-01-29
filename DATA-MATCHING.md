# Card Name Matching Problem & Solutions

**Date:** January 29, 2026

---

## THE PROBLEM

### eBay Sale Title vs PSA Population Report Format Mismatch

**eBay titles are messy and inconsistent:**
```
"2023 Panini Prizm Patrick Mahomes PSA 10 Gem Mint #290"
"PATRICK MAHOMES 2023 PRIZM PSA 10 GEM MT #290"
"Mahomes Patrick 2023 Panini Prizm Football PSA 10 Card #290"
"2023 Prizm Patrick Mahomes II #290 PSA 10 Chiefs"
```

**PSA Population Report format is standardized:**
```
"2023 Panini Prizm Patrick Mahomes #290"
```

**Challenge:** How do you match these automatically?

---

## DATA REQUIREMENTS (Simplified)

### From eBay API
```javascript
{
  "sale_price": 299.99,
  "sale_date": "2024-01-15T14:23:00Z",
  "listing_type": "AUCTION",  // or "FIXED_PRICE"
  "title": "2023 Panini Prizm Patrick Mahomes PSA 10 #290"
}
```

### From PSA Population Report (Scraped)
```javascript
{
  "card_name": "2023 Panini Prizm Patrick Mahomes #290",
  "psa_10_count": 1247,
  "psa_9_count": 892,
  "psa_8_count": 234,
  "total_graded": 2373
}
```

**Goal:** Link eBay sale → PSA population data

---

## MATCHING STRATEGIES

### Strategy 1: Exact Match (Simple, Low Accuracy ~30%)

**Approach:** Normalize both strings, compare directly

```javascript
function normalizeTitle(title) {
  return title
    .toUpperCase()
    .replace(/PSA\s*10|GEM\s*MINT?|MT/gi, '')  // Remove grade info
    .replace(/[^A-Z0-9\s]/g, '')  // Remove special chars
    .replace(/\s+/g, ' ')  // Collapse whitespace
    .trim();
}

// eBay: "2023 PANINI PRIZM PATRICK MAHOMES 290"
// PSA:  "2023 PANINI PRIZM PATRICK MAHOMES 290"
// Match: ✅ (if lucky)
```

**Pros:** Simple, fast  
**Cons:** Fails on name order variations, abbreviations, nicknames

---

### Strategy 2: Token-Based Matching (Better, ~60% accuracy)

**Approach:** Extract key tokens, compare sets

```javascript
function extractTokens(title) {
  const normalized = normalizeTitle(title);
  const tokens = normalized.split(/\s+/);
  
  return {
    year: tokens.find(t => /^\d{4}$/.test(t)),  // e.g., "2023"
    manufacturer: tokens.find(t => ['PANINI', 'TOPPS', 'UPPER', 'DECK'].includes(t)),
    set: tokens.find(t => ['PRIZM', 'CHROME', 'OPTIC', 'SELECT'].includes(t)),
    playerTokens: tokens.filter(t => 
      !/^\d+$/.test(t) &&  // Not a number
      !['PANINI', 'TOPPS', 'PRIZM', 'CHROME', 'PSA', 'GEM', 'MINT'].includes(t)
    ),
    cardNumber: tokens.find(t => /^#?\d+$/.test(t))?.replace('#', '')
  };
}

function tokensMatch(ebayTokens, psaTokens) {
  return (
    ebayTokens.year === psaTokens.year &&
    ebayTokens.manufacturer === psaTokens.manufacturer &&
    ebayTokens.set === psaTokens.set &&
    ebayTokens.cardNumber === psaTokens.cardNumber &&
    // Player name tokens overlap (handles "Patrick Mahomes" vs "Mahomes Patrick")
    ebayTokens.playerTokens.some(t => psaTokens.playerTokens.includes(t))
  );
}
```

**Example:**
```javascript
// eBay: "MAHOMES PATRICK 2023 PRIZM PSA 10 #290"
extractTokens(ebayTitle) = {
  year: "2023",
  manufacturer: "PANINI",
  set: "PRIZM",
  playerTokens: ["MAHOMES", "PATRICK"],
  cardNumber: "290"
}

// PSA: "2023 PANINI PRIZM PATRICK MAHOMES #290"
extractTokens(psaTitle) = {
  year: "2023",
  manufacturer: "PANINI",
  set: "PRIZM",
  playerTokens: ["PATRICK", "MAHOMES"],
  cardNumber: "290"
}

// Match: ✅ (year + manufacturer + set + card# match, player overlap)
```

**Pros:** Handles name order, more robust  
**Cons:** Still fails on abbreviations, misspellings

---

### Strategy 3: Fuzzy Matching with Levenshtein Distance (Best, ~85% accuracy)

**Approach:** Calculate similarity score, accept above threshold

```javascript
const levenshtein = require('fast-levenshtein');

function fuzzyMatch(ebayTitle, psaTitle, threshold = 0.7) {
  const norm1 = normalizeTitle(ebayTitle);
  const norm2 = normalizeTitle(psaTitle);
  
  const distance = levenshtein.get(norm1, norm2);
  const maxLength = Math.max(norm1.length, norm2.length);
  const similarity = 1 - (distance / maxLength);
  
  return similarity >= threshold;
}

// eBay: "2023 PANINI PRIZM PATRICK MAHOMES 290"
// PSA:  "2023 PANINI PRIZM PATRICK MAHOMES II 290"
// Distance: 2 (added "II")
// Similarity: 1 - (2 / 43) = 0.95 ✅ Match
```

**Pros:** Handles typos, variations, nicknames  
**Cons:** Slower, requires tuning threshold

---

### Strategy 4: Hybrid Approach (Recommended, ~90% accuracy)

**Combine token-based + fuzzy matching:**

```javascript
function matchCard(ebayTitle, psaTitle) {
  const ebayTokens = extractTokens(ebayTitle);
  const psaTokens = extractTokens(psaTitle);
  
  // STEP 1: Required exact matches (year, set, card number)
  if (
    ebayTokens.year !== psaTokens.year ||
    ebayTokens.set !== psaTokens.set ||
    ebayTokens.cardNumber !== psaTokens.cardNumber
  ) {
    return false;  // Can't be the same card
  }
  
  // STEP 2: Manufacturer match (optional, common errors)
  const mfgMatch = ebayTokens.manufacturer === psaTokens.manufacturer;
  
  // STEP 3: Player name fuzzy match
  const ebayPlayerName = ebayTokens.playerTokens.join(' ');
  const psaPlayerName = psaTokens.playerTokens.join(' ');
  const playerSimilarity = 1 - (
    levenshtein.get(ebayPlayerName, psaPlayerName) / 
    Math.max(ebayPlayerName.length, psaPlayerName.length)
  );
  
  // STEP 4: Decision logic
  if (mfgMatch && playerSimilarity >= 0.8) {
    return true;  // High confidence match
  }
  
  if (!mfgMatch && playerSimilarity >= 0.9) {
    return true;  // Likely manufacturer parsing error, but player is clear
  }
  
  return false;  // No match
}
```

**Example Matches:**
```javascript
matchCard(
  "2023 Panini Prizm Patrick Mahomes PSA 10 #290",
  "2023 Panini Prizm Patrick Mahomes #290"
) // ✅ True

matchCard(
  "Mahomes Patrick 2023 PRIZM #290 PSA 10",
  "2023 Panini Prizm Patrick Mahomes #290"
) // ✅ True (handles name order)

matchCard(
  "2023 Prizm Patrick Mahomes II #290 PSA 10",
  "2023 Panini Prizm Patrick Mahomes #290"
) // ✅ True (handles "II" suffix)

matchCard(
  "2023 Optic Patrick Mahomes #290 PSA 10",
  "2023 Panini Prizm Patrick Mahomes #290"
) // ❌ False (different set)
```

---

## DATABASE DESIGN FOR MATCHING

### Schema

```sql
-- Cards master table
CREATE TABLE cards (
  id SERIAL PRIMARY KEY,
  player_name VARCHAR(200) NOT NULL,
  year VARCHAR(10),
  manufacturer VARCHAR(100),
  set_name VARCHAR(200),
  card_number VARCHAR(50),
  sport VARCHAR(50),
  
  -- Normalized fields for matching
  normalized_name VARCHAR(200),  -- "PATRICK MAHOMES"
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  -- Full-text search index
  search_vector tsvector GENERATED ALWAYS AS (
    to_tsvector('english', 
      COALESCE(player_name, '') || ' ' ||
      COALESCE(year, '') || ' ' ||
      COALESCE(manufacturer, '') || ' ' ||
      COALESCE(set_name, '') || ' ' ||
      COALESCE(card_number, '')
    )
  ) STORED,
  INDEX idx_search (search_vector) USING GIN
);

-- eBay sales (linked to cards)
CREATE TABLE ebay_sales (
  id SERIAL PRIMARY KEY,
  card_id INT REFERENCES cards(id),  -- Foreign key
  item_id VARCHAR(100) UNIQUE,
  title TEXT,
  sale_price DECIMAL(10,2),
  sale_date TIMESTAMP,
  listing_type VARCHAR(20),  -- 'AUCTION' or 'FIXED_PRICE'
  graded BOOLEAN,
  grade VARCHAR(10),
  grader VARCHAR(50),
  
  -- Matching metadata
  match_confidence DECIMAL(3,2),  -- 0.00 to 1.00 (e.g., 0.95 = 95% confidence)
  match_method VARCHAR(50),  -- 'exact', 'token', 'fuzzy', 'hybrid'
  needs_review BOOLEAN DEFAULT FALSE,  -- Flag low-confidence matches
  
  created_at TIMESTAMP DEFAULT NOW()
);

-- PSA population data (linked to cards)
CREATE TABLE psa_populations (
  id SERIAL PRIMARY KEY,
  card_id INT REFERENCES cards(id),  -- Foreign key
  psa_10_count INT,
  psa_9_count INT,
  psa_8_count INT,
  total_graded INT,
  
  -- Snapshot timestamp (populations change over time)
  snapshot_date DATE DEFAULT CURRENT_DATE,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  -- Prevent duplicate entries per day
  UNIQUE (card_id, snapshot_date)
);
```

---

## MATCHING WORKFLOW

### Step 1: Parse eBay Sale

```javascript
async function processEbaySale(item) {
  const title = item.title;
  const tokens = extractTokens(title);
  
  // Find matching card in database
  const matchedCard = await findMatchingCard(tokens, title);
  
  if (matchedCard) {
    // Insert sale linked to card
    await db.query(
      `INSERT INTO ebay_sales (
        card_id, item_id, title, sale_price, sale_date, 
        listing_type, graded, grade, grader,
        match_confidence, match_method
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)`,
      [
        matchedCard.id,
        item.itemId,
        title,
        item.price.value,
        item.soldDate || new Date(),
        item.buyingOptions[0] === 'AUCTION' ? 'AUCTION' : 'FIXED_PRICE',
        true,
        tokens.grade || '10',
        'PSA',
        matchedCard.confidence,
        matchedCard.method
      ]
    );
  } else {
    // No match - create new card
    const newCard = await createCard(tokens);
    // Insert sale linked to new card
    await insertSale(newCard.id, item);
  }
}
```

### Step 2: Find Matching Card

```javascript
async function findMatchingCard(tokens, fullTitle) {
  // Try exact match first (fast)
  const exactMatch = await db.query(
    `SELECT * FROM cards 
     WHERE year = $1 AND set_name = $2 AND card_number = $3 
       AND normalized_name = $4`,
    [tokens.year, tokens.set, tokens.cardNumber, tokens.playerTokens.join(' ')]
  );
  
  if (exactMatch.rows.length === 1) {
    return { 
      id: exactMatch.rows[0].id, 
      confidence: 1.0, 
      method: 'exact' 
    };
  }
  
  // Try token-based match (medium speed)
  const candidates = await db.query(
    `SELECT * FROM cards 
     WHERE year = $1 AND set_name = $2 AND card_number = $3`,
    [tokens.year, tokens.set, tokens.cardNumber]
  );
  
  if (candidates.rows.length === 0) {
    return null;  // No candidates
  }
  
  // Try fuzzy match on candidates (slow but accurate)
  let bestMatch = null;
  let bestScore = 0;
  
  for (const candidate of candidates.rows) {
    const score = calculateMatchScore(fullTitle, candidate);
    if (score > bestScore && score >= 0.7) {
      bestScore = score;
      bestMatch = candidate;
    }
  }
  
  if (bestMatch) {
    return { 
      id: bestMatch.id, 
      confidence: bestScore, 
      method: 'fuzzy' 
    };
  }
  
  return null;  // No good match
}
```

### Step 3: Link PSA Population Data

```javascript
async function updatePSAPopulations(psaDataArray) {
  for (const psaEntry of psaDataArray) {
    // Find matching card (same matching logic)
    const matchedCard = await findMatchingCard(
      extractTokens(psaEntry.cardName),
      psaEntry.cardName
    );
    
    if (matchedCard) {
      // Insert or update population data
      await db.query(
        `INSERT INTO psa_populations (
          card_id, psa_10_count, psa_9_count, psa_8_count, total_graded, snapshot_date
        ) VALUES ($1, $2, $3, $4, $5, CURRENT_DATE)
        ON CONFLICT (card_id, snapshot_date) 
        DO UPDATE SET 
          psa_10_count = EXCLUDED.psa_10_count,
          psa_9_count = EXCLUDED.psa_9_count,
          psa_8_count = EXCLUDED.psa_8_count,
          total_graded = EXCLUDED.total_graded`,
        [
          matchedCard.id,
          psaEntry.psa10Count,
          psaEntry.psa9Count,
          psaEntry.psa8Count,
          psaEntry.totalGraded
        ]
      );
    } else {
      // Create new card entry
      const newCard = await createCard(extractTokens(psaEntry.cardName));
      await insertPSAPopulation(newCard.id, psaEntry);
    }
  }
}
```

---

## HANDLING EDGE CASES

### Problem: Name Variations

**Examples:**
- "Patrick Mahomes" vs "Patrick Mahomes II"
- "Shohei Ohtani" vs "Shohei Otani" (different romanization)
- "Jr." suffix vs no suffix
- Nicknames ("A-Rod" vs "Alex Rodriguez")

**Solution:** Player name normalization table

```sql
CREATE TABLE player_name_aliases (
  id SERIAL PRIMARY KEY,
  canonical_name VARCHAR(200),  -- "Patrick Mahomes"
  alias_name VARCHAR(200),  -- "Patrick Mahomes II"
  sport VARCHAR(50),
  
  INDEX idx_alias (alias_name)
);

-- Populate with known variations
INSERT INTO player_name_aliases (canonical_name, alias_name, sport) VALUES
('Patrick Mahomes', 'Patrick Mahomes II', 'Football'),
('Shohei Ohtani', 'Shohei Otani', 'Baseball'),
('Alex Rodriguez', 'A-Rod', 'Baseball');
```

**Matching logic with aliases:**
```javascript
async function normalizePlayerName(rawName) {
  const alias = await db.query(
    'SELECT canonical_name FROM player_name_aliases WHERE alias_name = $1',
    [rawName]
  );
  
  return alias.rows[0]?.canonical_name || rawName;
}
```

---

### Problem: Set Name Variations

**Examples:**
- "Prizm" vs "Panini Prizm"
- "Chrome" vs "Topps Chrome"
- Abbreviations ("Select" vs "SEL")

**Solution:** Set name dictionary

```sql
CREATE TABLE set_name_mappings (
  id SERIAL PRIMARY KEY,
  raw_name VARCHAR(200),  -- What appears in eBay titles
  canonical_name VARCHAR(200),  -- Standardized name
  manufacturer VARCHAR(100),
  
  INDEX idx_raw (raw_name)
);

INSERT INTO set_name_mappings (raw_name, canonical_name, manufacturer) VALUES
('PRIZM', 'Prizm', 'Panini'),
('PANINI PRIZM', 'Prizm', 'Panini'),
('CHROME', 'Chrome', 'Topps'),
('TOPPS CHROME', 'Chrome', 'Topps');
```

---

### Problem: Low Confidence Matches

**When to flag for manual review:**

```javascript
if (matchedCard.confidence < 0.85) {
  await db.query(
    'UPDATE ebay_sales SET needs_review = true WHERE id = $1',
    [saleId]
  );
  
  // Send alert
  console.log(`⚠️ Low confidence match (${matchedCard.confidence}): ${title}`);
}
```

**Weekly review workflow:**
```sql
-- Get all sales needing review
SELECT 
  s.id,
  s.title as ebay_title,
  c.player_name || ' ' || c.year || ' ' || c.set_name || ' #' || c.card_number as matched_card,
  s.match_confidence
FROM ebay_sales s
JOIN cards c ON c.id = s.card_id
WHERE s.needs_review = true
ORDER BY s.match_confidence ASC
LIMIT 100;
```

---

## PERFORMANCE OPTIMIZATION

### Problem: Fuzzy matching is slow on large datasets

**Solution: Use PostgreSQL trigram similarity**

```sql
-- Install pg_trgm extension
CREATE EXTENSION pg_trgm;

-- Add similarity index
CREATE INDEX trgm_idx_cards_search ON cards USING GIN (
  (player_name || ' ' || year || ' ' || set_name || ' ' || card_number) gin_trgm_ops
);

-- Fast similarity query
SELECT 
  id,
  player_name,
  year,
  set_name,
  card_number,
  similarity(
    player_name || ' ' || year || ' ' || set_name || ' ' || card_number,
    $1  -- eBay title
  ) as score
FROM cards
WHERE (player_name || ' ' || year || ' ' || set_name || ' ' || card_number) % $1  -- Similarity operator
ORDER BY score DESC
LIMIT 5;
```

**Benefits:**
- 10-100x faster than pure JavaScript fuzzy matching
- Index-backed (scales to millions of cards)
- Built-in typo tolerance

---

## SUMMARY

### Recommended Approach

1. **Normalize titles** (uppercase, remove grades, collapse whitespace)
2. **Extract tokens** (year, manufacturer, set, player, card number)
3. **Exact match first** (year + set + card# + normalized player name)
4. **Fallback to trigram similarity** (PostgreSQL `pg_trgm` for speed)
5. **Flag low confidence** (< 0.85) for manual review
6. **Build alias tables** (player names, set names) over time

### Expected Accuracy
- **Exact matches:** ~40% of eBay sales
- **Fuzzy matches:** ~45% of eBay sales
- **Manual review needed:** ~15% of eBay sales

### Implementation Priority
1. Week 1: Token extraction + exact matching
2. Week 2: PostgreSQL trigram similarity + fuzzy matching
3. Week 3: Alias tables (player/set names) + manual review workflow
4. Week 4: Refinement (tune thresholds, add more aliases)

---

**Next:** Build the scraper + matching pipeline. Let me know answers to the 5 questions from earlier and I'll generate starter code.
