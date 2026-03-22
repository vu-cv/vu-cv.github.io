---
layout: article
title: Elasticsearch – Full-text Search mạnh mẽ cho NodeJS
tags: [elasticsearch, search, nodejs, nestjs, full-text-search]
---
Elasticsearch là search engine phân tán, cực nhanh cho full-text search, analytics, và log aggregation. Khi PostgreSQL full-text search không đủ (tiếng Việt, fuzzy search, faceted search), Elasticsearch là lựa chọn tiêu chuẩn.

## 1. Cài đặt với Docker

```yaml
# docker-compose.yml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false   # Tắt security cho dev
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200

volumes:
  es_data:
```

## 2. Kết nối từ NestJS

```bash
npm install @elastic/elasticsearch
```

```typescript
// src/search/elasticsearch.service.ts
import { Injectable, OnModuleInit, Logger } from '@nestjs/common';
import { Client } from '@elastic/elasticsearch';

@Injectable()
export class ElasticsearchService implements OnModuleInit {
  private readonly logger = new Logger(ElasticsearchService.name);
  client: Client;

  constructor() {
    this.client = new Client({
      node: process.env.ELASTICSEARCH_URL ?? 'http://localhost:9200',
      auth: process.env.ES_USERNAME
        ? { username: process.env.ES_USERNAME, password: process.env.ES_PASSWORD! }
        : undefined,
    });
  }

  async onModuleInit() {
    try {
      const info = await this.client.info();
      this.logger.log(`Connected to Elasticsearch ${info.version.number}`);
    } catch (error) {
      this.logger.error('Failed to connect to Elasticsearch:', error.message);
    }
  }
}
```

## 3. Tạo Index (Schema)

```typescript
// src/search/products.index.ts
export const PRODUCTS_INDEX = 'products';

export const productsMapping = {
  mappings: {
    properties: {
      id: { type: 'keyword' },
      name: {
        type: 'text',
        analyzer: 'vi_analyzer',   // Analyzer tiếng Việt
        fields: {
          keyword: { type: 'keyword' }, // Exact match
          suggest: { type: 'completion' }, // Autocomplete
        },
      },
      description: { type: 'text', analyzer: 'vi_analyzer' },
      category: { type: 'keyword' },
      brand: { type: 'keyword' },
      price: { type: 'float' },
      stock: { type: 'integer' },
      tags: { type: 'keyword' },
      rating: { type: 'float' },
      viewCount: { type: 'integer' },
      createdAt: { type: 'date' },
      published: { type: 'boolean' },
    },
  },
  settings: {
    analysis: {
      analyzer: {
        vi_analyzer: {
          type: 'custom',
          tokenizer: 'standard',
          filter: ['lowercase', 'asciifolding'], // asciifolding: bỏ dấu
        },
      },
    },
    number_of_shards: 1,
    number_of_replicas: 0, // 0 cho dev, 1 cho prod
  },
};

// Tạo index
async function createIndex(esService: ElasticsearchService) {
  const exists = await esService.client.indices.exists({ index: PRODUCTS_INDEX });
  if (!exists) {
    await esService.client.indices.create({
      index: PRODUCTS_INDEX,
      ...productsMapping,
    });
  }
}
```

## 4. Index Documents

```typescript
// src/search/products-search.service.ts
@Injectable()
export class ProductsSearchService {
  constructor(private esService: ElasticsearchService) {}

  // Index một product
  async indexProduct(product: Product): Promise<void> {
    await this.esService.client.index({
      index: PRODUCTS_INDEX,
      id: product.id,
      document: {
        id: product.id,
        name: product.name,
        description: product.description,
        category: product.category,
        brand: product.brand,
        price: product.price,
        stock: product.stock,
        tags: product.tags,
        rating: product.rating,
        published: product.published,
        createdAt: product.createdAt,
      },
    });
  }

  // Bulk index nhiều products
  async bulkIndex(products: Product[]): Promise<void> {
    const operations = products.flatMap(p => [
      { index: { _index: PRODUCTS_INDEX, _id: p.id } },
      {
        id: p.id, name: p.name, description: p.description,
        category: p.category, price: p.price, stock: p.stock,
        rating: p.rating, published: p.published,
      },
    ]);

    const result = await this.esService.client.bulk({ operations });
    if (result.errors) {
      const errors = result.items.filter(i => i.index?.error);
      this.logger.error('Bulk index errors:', errors);
    }
  }

  // Xóa document
  async deleteProduct(id: string): Promise<void> {
    await this.esService.client.delete({ index: PRODUCTS_INDEX, id });
  }
}
```

## 5. Tìm kiếm nâng cao

```typescript
async search(params: {
  query?: string;
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  brands?: string[];
  minRating?: number;
  sortBy?: 'price_asc' | 'price_desc' | 'rating' | 'newest' | 'relevance';
  page?: number;
  limit?: number;
}): Promise<{ products: any[]; total: number; aggregations: any }> {
  const { query, category, minPrice, maxPrice, brands, minRating, sortBy = 'relevance', page = 1, limit = 20 } = params;

  // Build query
  const must: any[] = [{ term: { published: true } }];
  const filter: any[] = [];

  if (query) {
    must.push({
      multi_match: {
        query,
        fields: ['name^3', 'description', 'tags^2', 'brand'],
        type: 'best_fields',
        fuzziness: 'AUTO',        // Cho phép lỗi chính tả
        operator: 'and',
      },
    });
  }

  if (category) filter.push({ term: { category } });
  if (brands?.length) filter.push({ terms: { brand: brands } });
  if (minRating) filter.push({ range: { rating: { gte: minRating } } });
  if (minPrice !== undefined || maxPrice !== undefined) {
    filter.push({ range: { price: { gte: minPrice, lte: maxPrice } } });
  }

  const sort: any[] = {
    relevance: [{ _score: 'desc' }],
    price_asc: [{ price: 'asc' }],
    price_desc: [{ price: 'desc' }],
    rating: [{ rating: 'desc' }],
    newest: [{ createdAt: 'desc' }],
  }[sortBy];

  const result = await this.esService.client.search({
    index: PRODUCTS_INDEX,
    from: (page - 1) * limit,
    size: limit,
    query: {
      bool: { must, filter },
    },
    sort,
    // Facets / Aggregations
    aggs: {
      categories: { terms: { field: 'category', size: 20 } },
      brands: { terms: { field: 'brand', size: 20 } },
      price_range: {
        range: {
          field: 'price',
          ranges: [
            { to: 100000 },
            { from: 100000, to: 500000 },
            { from: 500000, to: 1000000 },
            { from: 1000000 },
          ],
        },
      },
      avg_rating: { avg: { field: 'rating' } },
    },
    highlight: {
      fields: { name: {}, description: { fragment_size: 150 } },
      pre_tags: ['<mark>'],
      post_tags: ['</mark>'],
    },
  });

  return {
    products: result.hits.hits.map(hit => ({
      ...hit._source,
      score: hit._score,
      highlight: hit.highlight,
    })),
    total: (result.hits.total as any).value,
    aggregations: result.aggregations,
  };
}
```

## 6. Autocomplete / Suggest

```typescript
async suggest(prefix: string): Promise<string[]> {
  const result = await this.esService.client.search({
    index: PRODUCTS_INDEX,
    suggest: {
      product_suggest: {
        prefix,
        completion: {
          field: 'name.suggest',
          size: 8,
          fuzzy: { fuzziness: 1 },
        },
      },
    },
    _source: false,
  });

  return result.suggest!.product_suggest[0].options.map(o => o.text);
}
```

## 7. Sync DB với Elasticsearch

```typescript
// Đồng bộ khi có thay đổi trong DB
@OnEvent('product.created')
async onProductCreated(event: ProductCreatedEvent) {
  await this.productsSearchService.indexProduct(event.product);
}

@OnEvent('product.updated')
async onProductUpdated(event: ProductUpdatedEvent) {
  await this.productsSearchService.indexProduct(event.product);
}

@OnEvent('product.deleted')
async onProductDeleted(event: ProductDeletedEvent) {
  await this.productsSearchService.deleteProduct(event.productId);
}
```

## 8. Kết luận

- **Mapping**: Định nghĩa đúng kiểu dữ liệu và analyzer — ảnh hưởng lớn đến search quality
- **`asciifolding`**: Cho phép tìm "ao" ra "áo" — quan trọng cho tiếng Việt
- **`fuzziness: AUTO`**: Chịu lỗi chính tả — user gõ "điên thoại" → ra "điện thoại"
- **Aggregations**: Faceted search (filter theo category, brand, price range) trong 1 query
- **Sync**: Dùng Event Emitter để sync DB → Elasticsearch khi có thay đổi

Elasticsearch giải quyết search phức tạp mà SQL không làm được — nhưng tốn resource hơn. Dùng khi có > 100k documents hoặc cần fuzzy/tiếng Việt.
