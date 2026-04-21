# Valkey Vector Search Demo

Full-text search + vector search (KNN) demo using Amazon ElastiCache for Valkey 9.0 with synthetic e-commerce data.

## Overview

This project demonstrates Valkey's search capabilities:
- **Full-text search** with prefix and fuzzy matching
- **TAG filtering** for exact categorical matches
- **NUMERIC range filters** for price, rating, stock
- **Vector similarity search** (KNN) using HNSW with cosine distance
- **Hybrid queries** combining text, tags, numerics, and vector search

## Architecture

```
┌─────────────────┐     ┌──────────────────────────┐     ┌─────────────────────────┐
│  Load Script    │────▶│  Amazon Bedrock           │────▶│  ElastiCache Valkey 9.0 │
│  (Python)       │     │  Titan Embed Text v2      │     │  idx:products_demo      │
│                 │     │  1024-dim embeddings      │     │  10K products           │
└─────────────────┘     └──────────────────────────┘     └─────────────────────────┘
```

## Prerequisites

- Python 3.9+
- AWS credentials with Bedrock access
- Network access to your ElastiCache Valkey cluster
- Required packages: `redis`, `boto3`

```bash
pip install redis boto3
```

## Dataset

`data/products_synthetic_10k.json` — 10,000 synthetic e-commerce products across 10 categories:

| Field | Type | Description |
|-------|------|-------------|
| `product_id` | string | Unique identifier (PROD-00001..PROD-10000) |
| `title` | string | Product name |
| `description` | string | 1-3 sentence description |
| `brand` | string | Brand name (~200 real brands) |
| `category` | string | Pipe-separated hierarchy (e.g. "Electronics\|Headphones\|Wireless") |
| `color` | string/null | Product color (~30% null) |
| `price` | float | Price in USD (5.99–2999.99) |
| `rating` | float | Customer rating (1.0–5.0) |
| `stock` | int | Units in stock (0=out of stock, ~5% of products) |
| `bullet_points` | string | Key features separated by " \| " |

**Categories:** Electronics, Clothing, Home & Kitchen, Sports & Outdoors, Beauty, Toys & Games, Books, Automotive, Garden & Outdoor, Office Supplies (1,000 each)

## Index Schema

```
FT.CREATE idx:products_demo
  ON HASH
  PREFIX 1 demo:
  SCHEMA
    title TEXT
    description TEXT
    brand TAG
    category TAG SEPARATOR |
    color TAG
    price NUMERIC SORTABLE
    rating NUMERIC SORTABLE
    stock NUMERIC SORTABLE
    embedding VECTOR HNSW 6
      TYPE FLOAT32
      DIM 1024
      DISTANCE_METRIC COSINE
```

## Usage

### 1. Load data into Valkey

Edit the connection settings in `scripts/load_index.py`:

```python
VALKEY_HOST = "your-cluster-endpoint.cache.amazonaws.com"
VALKEY_PORT = 6379
BEDROCK_REGION = "eu-west-1"  # Region with Bedrock Titan access
```

Run:

```bash
python scripts/load_index.py
```

This will:
1. Create the `idx:products_demo` index
2. Generate embeddings via Amazon Bedrock Titan Embed Text v2 (1024 dimensions)
3. Load all 10,000 products with their embeddings (~10 minutes)

### 2. Run demo queries

```bash
python scripts/demo_queries.py
```

## Search Query Examples

### Prefix Search

Find all products with titles starting with "wire":

```
FT.SEARCH idx:products_demo "@title:wire*" RETURN 3 title brand price LIMIT 0 5 DIALECT 2
```

Result: 124 matches (wireless, wired, etc.)
```
[HP] HP Notebook - Wireless              $362.60
[Acer] Wireless Cable by Acer            $20.80
[Apple] Apple Wireless Hub               $84.76
[Sony] Sony Wireless Keyboard            $181.16
```

### Fuzzy Search (typo tolerance)

Single edit distance — corrects "headphoens" → "headphones":

```
FT.SEARCH idx:products_demo "@title:%headphoens%" RETURN 3 title brand price LIMIT 0 5 DIALECT 2
```

Double edit distance — corrects "hedphones" → "headphones":

```
FT.SEARCH idx:products_demo "@title:%%hedphones%%" RETURN 3 title brand price LIMIT 0 5 DIALECT 2
```

Result: 65 matches
```
[Dell] Dell Smart Headphones             $1111.11
[Razer] HDR Headphones by Razer          $907.51
[Anker] 4K Headphones by Anker           $141.48
[HyperX] HyperX Compact Headphones       $26.67
```

### Text + Tag Filter

Full-text search combined with brand filtering:

```
FT.SEARCH idx:products_demo "@title:wireless @brand:{Sony|Bose}" RETURN 3 title brand price LIMIT 0 5 DIALECT 2
```

Result: 6 matches
```
[Sony] Sony Wireless Keyboard            $181.16
[Bose] Bose Wireless Power Bank          $1030.49
[Sony] Wireless Earbuds by Sony           $163.92
```

### Text + Numeric Range

Keyboards priced between $20-$100:

```
FT.SEARCH idx:products_demo "@title:keyboard @price:[20 100]" RETURN 4 title brand price rating LIMIT 0 5 DIALECT 2
```

Result: 26 matches
```
[LG] LG Keyboard - Compact              $65.22  ★3.8
[Sennheiser] Sennheiser 256GB Keyboard   $20.86  ★4.4
[Logitech] Logitech 256GB Keyboard       $94.89  ★3.4
[HP] HP Bluetooth Keyboard               $37.60  ★4.1
```

### Tag + Numeric (Multi-filter)

Electronics between $100-$300 with rating ≥ 4.5:

```
FT.SEARCH idx:products_demo "@category:{Electronics} @price:[100 300] @rating:[4.5 5.0]" RETURN 4 title brand price rating LIMIT 0 5 DIALECT 2
```

Result: 67 matches
```
[HyperX] 4K Keyboard by HyperX          $160.58  ★4.5
[Philips] Philips Monitor - Premium      $295.37  ★5.0
[Sony] Sony Premium Earbuds              $119.55  ★4.9
```

### Full Combo (Text + Tag + Numeric)

Organic beauty products, $10-$50, rating ≥ 4.0:

```
FT.SEARCH idx:products_demo "@title:organic @category:{Beauty} @price:[10 50] @rating:[4.0 5.0]" RETURN 4 title brand price rating LIMIT 0 5 DIALECT 2
```

Result: 19 matches
```
[OGX] OGX Organic Shampoo               $14.04  ★4.9
[Olay] Olay Organic Hair Oil             $40.82  ★4.0
[Garnier] Garnier Eye Cream - Organic    $43.26  ★4.6
```

### Negation

Laptops NOT from Apple, $500-$1500:

```
FT.SEARCH idx:products_demo "@title:laptop -@brand:{Apple} @price:[500 1500]" RETURN 4 title brand price rating LIMIT 0 5 DIALECT 2
```

Result: 14 matches
```
[Philips] Philips Laptop Stand - Bluetooth  $1115.87  ★4.3
[Samsung] Samsung Laptop Stand - Portable   $585.67   ★3.1
[Acer] Acer Compact Laptop Stand            $569.93   ★4.0
```

### Vector Search (KNN)

Semantic similarity search using embeddings:

```
FT.SEARCH idx:products_demo "*=>[KNN 5 @embedding $vec AS score]"
  PARAMS 2 vec <embedding_bytes>
  RETURN 4 title brand price score
  DIALECT 2
```

Query: "premium wireless noise cancelling headphones"
```
0.3684  [Asus] Wireless Headphones by Asus          $69.90
0.3872  [Samsung] Noise-Cancelling Headphones       $857.92
0.4397  [Sony] Premium Headphones by Sony           $351.74
0.4710  [Razer] Portable Headphones by Razer        $26.33
```

### Hybrid: Vector + Filters

KNN search with pre-filtering (price and rating constraints):

```
FT.SEARCH idx:products_demo "@price:[50 150] @rating:[4.0 5.0]=>[KNN 5 @embedding $vec AS score]"
  PARAMS 2 vec <embedding_bytes>
  RETURN 5 title brand price rating score
  DIALECT 2
```

Query: "lightweight running shoes for marathon"
```
0.6671  [The North Face] Lightweight Shorts         $144.82  ★4.1
0.6706  [Ralph Lauren] Slim-Fit Running Shoes       $135.48  ★4.0
0.6745  [Ralph Lauren] Slim-Fit Running Shoes       $115.04  ★4.3
0.6763  [Levi's] Levi's Lightweight Sneakers        $113.11  ★4.6
0.6868  [Columbia] Columbia Running Shoes - Stretch  $65.81   ★4.0
```

## Limitations & Notes

- **Suffix search** (`*phones`) is NOT supported on TEXT fields — requires TAG with `WITHSUFFIXTRIE`
- **WEIGHT clause** on TEXT fields only supports value `1.0` in ElastiCache Valkey 9.0
- **SORTBY** is not supported in `FT.SEARCH` — KNN results are pre-sorted by score
- Vector scores are **cosine distance** (lower = more similar, range 0–2)
- Embeddings are generated via Amazon Bedrock Titan Embed Text v2 (1024 dimensions)

## Cost Estimate

- **Bedrock embeddings:** ~$0.02 per 1M input tokens (~$0.10 for 10K products)
- **ElastiCache cache.t4g.medium:** ~$0.068/hour ($49/month)
- **EC2 t4g.micro (loader):** ~$0.0084/hour (negligible, terminate after loading)

## License

Dataset is synthetic. Code is provided as-is for demonstration purposes.
