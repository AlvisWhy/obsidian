### Full-text queries

Measurement
### Precision[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/introduction-to-search-request#Precision)

**Precision** is a measure of the accuracy of the search results. It is calculated as the number of relevant documents returned by the search divided by the total number of documents returned.

### Recall[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/introduction-to-search-request#Recall)

**Recall** is a measure of the completeness of the search results. It is calculated as the number of relevant documents returned by the search divided by the total number of relevant documents that exist in the index.

### Term frequency (TF)[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/introduction-to-search-request#Term-frequency-TF)

**Term frequency** is the frequency of occurrence of the search term(s) in the document—in other words, the number of times a term appears in a document. A document with a high term frequency is considered more relevant to the search query.

### Inverse document frequency (IDF)[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/introduction-to-search-request#Inverse-document-frequency-IDF)

The **inverse document frequency** is a statistical measure that determines the importance of a term or word in a document, based on how frequently it occurs across a collection of documents.

The **IDF value** of a term measures how rare or common the term is in the collection of documents. The **IDF factor** reduces the weight of a term that occurs frequently in a set of documents, while increasing the weight of a term that appears rarely.

### BM25 formula[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/introduction-to-search-request#BM25-formula)

**Best Match 25 (BM25)** is a ranking function employed in Elasticsearch to determine the relevance score between a document and a search query. It incorporates the key factors we previously discussed: term frequency (TF), inverse document frequency (IDF), and field length.

## Match query syntax
```
GET my_index_name/_search
{
  "query": {
    "match": {
      "my_field_name": "my_search_query"
    }
  }
}
```

The `analyzer` parameter is used to specify the analyzer that should be applied to a specific field during the indexing and searching processes.

The `operator` parameter in the match query provides control over the logical operation performed when multiple terms (tokens) are generated from the search text.
OR：Could contain either. AND: must contain all.
```
GET book/_search
{
  "query": {
    "match": {
      "description": {
        "query":"classic novel",
        "operator": "or"
      }
    }
  }
}

```
### `minimum_should_match`
specify the minimum number of optional terms (tokens) that must match for a document to be considered a match.
```
GET /book/_search 
{
  "query":{
    "match": {
      "description": {
        "query": "art history and its methodologies",
        "minimum_should_match": "75%"
      }
    }
  }
}
```
## Match phrase query[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/full-text-query-exploring-more-queries#Match-phrase-query)
It is useful when we want to find documents that contain a specific phrase in a specific order.
**consecutive and same order**

## Match phrase prefix query

The match phrase prefix query is particularly useful when looking for documents that match a specific phrase while accommodating variations at the end of the phrase. It is commonly used in scenarios where partial phrases or the autocomplete functionality are needed.
```
{ "query": { "match_phrase_prefix": { "title": "Learning Elas" } } }
```
## Multi-match query[#](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/full-text-query-exploring-more-queries#Multi-match-query)
Search for matches among multiple fields
```
GET book/_search
{
  "query": {
    "multi_match": {
      "query": "novel",
      "fields": ["title", "description"]
    }
  }
}
```

## Term query[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/term-level-queries-term-query#Term-query)

A term query is a type of query used to search for documents containing an exact term or value in a specific field. It is designed for searching `keyword` fields. No processing for search keys.

## Range query syntax[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/term-level-queries-range-query#Range-query-syntax)

To perform a `range` query, we can send a `GET` request to the `_search` endpoint of the target index.
- `gt`: Greater than
- `gte`: Greater than or equal to
- `lt`: Less than
- `lte`: Less than or equal to
```
GET my_index_name/_search
{
  "query": {
    "range": {
      "my_field_name": {
        "gt": 10,
        "lt": 100
      }
    }
  }

```
```
"terms": {
  "field": "year",
  "size": 100,
  "order": {
    "_key": "asc"
  }
}
```
Elasticsearch employs the concept of term ranges within the inverted index, not only for numeric values, but also for other types, such as dates. This indexing technique significantly enhances search efficiency, even with extensive datasets.

# More Term-Level Queries

- Prefix query
    
- Wildcard query
    
- Regex query

## Fuzzy search[#](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/fuzzy-search#Fuzzy-search)

using an **edit distance algorithm** that represents the minimum number of operations required to transform one term into another.

```
GET my_index_name/_search
{
  "query": {
    "match": {
      "my_field_name": {
        "query": "search_term",
        "fuzziness": edit distance value,
        "prefix_length": prefix length value (optional),
        "fuzzy_transpositions": true/false (optional)
      }
    }
  }
}
```

### Fuzzy search with the term-level query

The fuzzy query operates similar to a term query. It does not analyze the search query and treats it as a single term.

```
GET my_index_name/_search
{
  "query": {
    "fuzzy": {
      "my_field_name": {
        "value": "search_term",
        "fuzziness": edit distance value,
        "prefix_length": prefix length value (optional),
        "fuzzy_transpositions": true/false (optional)
      }
    }
  }
}
```

## Boolean query[#](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/compound-queries-boolean-query#Boolean-query)

- `must`: The `must` clause specifies criteria that must be satisfied by the matching documents. In other words, the documents must match the conditions specified in the `must` clause to be considered search results.
    
- `must_not`: The `must_not` clause excludes documents that match the specified conditions. It sets criteria that must not be satisfied by the documents for them to be considered valid matches.
    
- `should`: The `should` clause specifies the criteria that are desirable but not mandatory for the matching documents. Documents that satisfy the `should` conditions will have a higher relevance score, making them more likely to appear in the search results. However, documents that do not meet the `should` conditions can still be considered valid matches.
    
- `filter`: The `filter` clause is similar to the `must` clause in terms of mandatory conditions. However, unlike the `must` clause, the `filter` clause does not contribute to score calculation. It is primarily used for filtering documents based on specific criteria, such as exact matches or range conditions, without affecting the relevance ranking.

```
GET my_index_name/_search
{
  "query": {
    "bool": {
      "must": [ /* array of must queries */ ],
      "should": [ /* array of should queries */ ],
      "must_not": [ /* array of must_not queries */ ],
      "filter": [ /* array of filter queries */ ]
    }
  }
}

```


## Aggregation[#](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/introduction-to-aggregation#Aggregation)

### Metrics aggregations[#](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/introduction-to-aggregation#Metrics-aggregations)
**Metrics aggregations** in Elasticsearch are used to calculate various metrics or statistics on a numeric field in a collection of documents.

- **Sum aggregation:** It calculates the sum of a numeric field across all documents matching the query.
```
GET /product/_search
{
  "size":0,
  "query": {
    "term": {
      "brand": "Apple"
    }
  },
  "aggs": {
    "apple_prices_sum": {
      "sum": {
        "field": "price"
      }
    }
  }
}
    
```
- **Avg aggregation:** It computes the average of a numeric field across the matching documents.
    
- **Min aggregation:** It finds the minimum value of a numeric field among the documents.
    
- **Max aggregation:** It determines the maximum value of a numeric field among the documents.
    
- **Stats aggregation:** It provides a collection of basic statistics about a numeric field, including count, sum, average, min, and max.
```
GET product/_search
{
  "size": 0,
  "aggs": {
    "price_statistics": {
      "stats": {
        "field": "price"
      }
    }
  }
}    
```
- **Extended stats aggregation:** It is similar to the stats aggregation, but it includes additional statistical information such as standard deviation and variance.
    
- **Percentiles aggregation:** It computes one or more percentiles (e.g., 25th, 50th, and 75th) for a numeric field.
    
- **Cardinality aggregation:** It estimates the distinct count of values in a numeric or keyword field.
    
- **Value count aggregation:** It counts the number of non-null values in a numeric or keyword field.
    
- **The** `top_hits` **metric aggregator:** It keeps track of the most relevant document being aggregated. This aggregator is intended to be used as a sub-aggregator so that the top matching documents can be aggregated per bucket.

### Bucket aggregations[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/introduction-to-aggregation#Bucket-aggregations)

**Bucket aggregations** in Elasticsearch are a powerful method of grouping documents into buckets based on specific criteria.

- Terms aggregation
	**Term aggregation** groups documents based on unique terms found within a specified field.it
	When conducting term aggregation on a `text` field, a bucket is created for each term present in the inverted index, deriving from the index generated through analysis by the analyzer.
```
GET product/_search
{
  "size": 0,
  "aggs": {
    "products_per_seller": {
      "terms": {
        "field": "seller"
      }
    }
  }
}
```
- Range aggregation
	**Range aggregation** is a type of bucket aggregation that allows us to group documents into buckets based on specified ranges of values from a numeric or date field.
```
GET product/_search
{
  "size": 0,
  "query": {
    "term": {
      "category": "Electronics"
    }
  },
  "aggs": {
    "rating_ranges": {
      "range": {
        "field": "rating",
        "ranges": [
          { "to": 1 },
          { "from": 1, "to": 2 },
          { "from": 2, "to": 3 },
          { "from": 3, "to": 4 },
          { "from": 4 }
        ]
      }
    }
  }
}
```

- Histogram aggregation
	Histogram aggregation is somewhat similar to range aggregation, but instead of specifying explicit ranges, we define the interval size, and Elasticsearch automatically creates ranges (bins) for us.
	
```
GET product/_search
{
  "size": 0,
  "aggs": {
    "price_histogram": {
      "histogram": {
        "field": "price",
        "interval": 500
      }
    }
  }
}
```

**Pipeline aggregation** in Elasticsearch allows us to perform additional calculations and computations on the results of other aggregations.

```
{
/* Optional: Setting size to 0 ensures 
    the search results won't include individual documents.*/
  "size": 0,
  "query": {
    // Your query for filtering data (optional)
  },
  "aggs": {
    "aggregation_name": {
      "aggregation_type": {
        // Aggregation options and parameters
      }
    },
  }
}

POST /products/_search
{
  "size": 0,
  "aggs": {
    "unique_brand_count": {
      "cardinality": {
        "field": "brand.keyword"
      }
    }
  }
}
```

## Combined aggregation
```
GET my_index/_search
{
  "size": 0,
  "aggs": {
    "bucket_name": {
      "bucket_aggregation_type": {
        "field": "field_name",
        "size": bucket_size,
        ... (other bucket aggregation parameters),
      
        "aggs": {
          "metric_name": {
            "metric_aggregation_type": {
              "field": "field_name"
              ... (other metric aggregation parameters)
            }
          },
          ... (additional metric aggregations)
        }
      }
    }
  }
}

```

### Pipeline Aggregation

### Sibling aggregation
A **sibling aggregation** within Elasticsearch’s aggregation receives the output of another aggregation at the same level and uses it to calculate a new aggregation result. 操作新产生的field的层级在bucket层。
- **Average bucket aggregation:** This aggregation computes the average value from the buckets generated by a bucket aggregation.
    
- **Max bucket aggregation:** This aggregation calculates the maximum value from the buckets created by a bucket aggregation.
    
- **Min bucket aggregation:** This aggregation determines the minimum value from the buckets produced by a bucket aggregation.
    
- **Stats bucket aggregation:** This aggregation provides statistical insights, including the count, sum, average, minimum, and maximum, for the buckets generated by a bucket aggregation.

### Parent aggregations[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/pipeline-aggregations#Parent-aggregations)

**A parent aggregation** is a group of pipeline aggregations that receives the results of its parent aggregation’s output and can generate additional buckets or perform fresh aggregations to augment existing buckets.操作新产生的field的层级在每个bucket层的内部。

- Bucket sort aggregation
- Bucket selector aggregation
- Bucket script aggregation
### Pipeline aggregation syntax[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/pipeline-aggregations#Pipeline-aggregation-syntax)

When employing pipeline aggregations in Elasticsearch, the key concept is the use of `buckets_path`, which refers to the paths of the aggregations we intend to use within our pipeline. The aggregation name itself serves as the path, and for nested aggregations, we use a format like `aggregationA>aggregationB`.
```
GET my_index/_search
{
  "size": 0,
  "aggs": {
    "my_aggregation_name": {
      "bucket_aggregation_type": {
        "field": "field_name",
        ... (other bucket aggregation parameters),
      
        "aggs": {
          "parnent_pipeline_agg_name": {
              "parnent_pipeline_agg_type": {
                "buckets_path": "aggregation_path", // buckets_path could be more than one value
                ... (other parnent pipeline aggregation parameters),
              }
            }
          ... (additional aggregations)
        }
      }
    },
    "slibing_pipeline_agg_name": {
      "slibing_pipeline_agg_type": {
        "buckets_path": "aggregation_path", // buckets_path could be more than one value
        ... (other slibing pipeline aggregation parameters),
      }
    }
  }
}
```

## Aggregation操作符

```
Aggregation
├── Bucket Aggregation
│     terms
│     date_histogram
│     histogram
│
├── Metric Aggregation
│     sum
│     avg
│     max
│     min
│
└── Pipeline Aggregation
      max_bucket
      avg_bucket
      sum_bucket
      derivative
      bucket_sort
      bucket_selector
```
```
GET product_orders/_search
{
  "size": 0,
  "aggs": {
    "average_sales_per_category": {
      "terms": {
        "field": "product_category"  
      },
      "aggs": {
        "avg_sales": {
          "avg": {
            "field": "sales"
          }
        },
        "average_above_140": {
          "bucket_selector": {
            "buckets_path": {
              "averageSales": "avg_sales"
            },
            "script": "params.averageSales >= 140"
          }
        },
        "ascending_order": {
          "bucket_sort": {
            "sort": [
              { "avg_sales": { "order": "asc" } }
            ]
          }
        }
      }
    }
  }
}
```
`params` 是 Painless 脚本中的参数对象（一个 Map）。在 `bucket_selector` 中，`buckets_path` 会把前面聚合得到的指标值映射成 `params` 中的变量

`buckets_path` 就像 Pipeline Aggregation 的"输入参数"。它负责告诉 Elasticsearch 从哪个 Aggregation 读取结果；对于带脚本的 Pipeline Aggregation（如 `bucket_selector`、`bucket_script`），还会把这些结果映射为 `params` 中可使用的变量。

| `buckets_path` 写法                     | 作用                          |
| ------------------------------------- | --------------------------- |
| `"avg_sales"`                         | 引用一个聚合结果（路径）                |
| `"category>avg_sales"`                | 引用嵌套聚合结果                    |
| `{ "sales": "avg_sales" }`            | 把聚合结果绑定为脚本变量 `params.sales` |
| `{ "sales": "...", "orders": "..." }` | 给多个聚合结果分别绑定变量               |
# Nested Bucket Aggregations

```
GET /our_index/_search
{
  "size": 0,
  "aggs": {
    "outer_agg_name": {
      "outer_agg_type": {
        "outer_agg_parameters": "..."
      },
      "aggs": {
        "inner_agg_name": {
          "inner_agg_type": {
            "inner_agg_parameters": "..."
          }
        }
        // Add more nested aggregations as needed
      }
    }
  }
}
```

```
GET /product_orders/_search
{
  "size": 0,
  "aggs": {
    "daily_order_count": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "day"
      },
      "aggs": {
        "orders_per_category": {
          "terms": {
            "field": "product_category"
          }
        }
      }
    }
  }
}
```

```
GET movie/_search
{
  "size": 0,
  "aggs": {
    "rate_info": {
      "terms": {
        "field": "rating"
      },
      "aggs": {
        "top_rated_movie": {
          "top_hits": {
            "size": 1,
            "sort": [
              {
                "boteAvg": {
                  "order": "desc"
                }
              }
            ],
            // Using source to return values with the result
            "_source": [
              "title",
              "boteAvg",
              "rating"
            ]
          }
        }
      }
    }
  }
}
```

`top_hits` 是一种**返回文档（Document）的 Aggregation**。

可以理解为：

> **在每个 Bucket 中，按照指定规则排序，然后返回前 N 条文档。**


如果结果需要"每条文档 + 一个聚合值"一起计算（例如 `budget - average_budget`），Elasticsearch Aggregation 无法直接实现，因为 Pipeline 不能访问 Document，而 Document Script 也不能访问 Aggregation。通常需要两次查询，或者在应用层计算。