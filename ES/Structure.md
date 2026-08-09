shard
- **Improved performance:** By distributing data across multiple shards, search and indexing operations can be parallelized, resulting in improved performance.
    
- **Increased capacity:** Sharding allows Elasticsearch to scale horizontally, allowing it to handle larger volumes of data.

### Replicas[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/architecture-of-elasticsearch#Replicas)

A **replica** is a copy of a shard. When a shard has one or more replicas, if the primary shard fails or becomes unavailable, one of the replicas can be promoted to primary.

Replicas can also be used to scale search performance. For example, when a search request is made, it can be executed on all shards (including replicas) in parallel.

### Mapping
Dynamic Mapping
Some example dynamic mapping rules

|                |                                                                                                                       |                                   |     |
| -------------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------------- | --- |
| JSON Data Type | Specifications                                                                                                        | Elasticsearch Mapping             |     |
| `boolean`      | **-**                                                                                                                 | `boolean`                         |     |
| `long`         | **-**                                                                                                                 | `long`                            |     |
| `double`       | **-**                                                                                                                 | `double`                          |     |
| `object`       | **-**                                                                                                                 | `object`                          |     |
| `array`        | Depends on the first non-null value in the `array` to determine its type                                              | `array`                           |     |
| `string`       | Strings that contain dates such as `"2022-01-01"` will be mapped as the `date` datatype.                              | `date`                            |     |
| `string`       | Strings that contain numbers such as `"132"` will be mapped as the `long` datatype.                                   | `long`                            |     |
| `string`       | Strings that contain normal text such as `"Fifa worldcup 2022"` will be mapped as the `text` and `keyword`data types. | `text` with a `keyword` sub-field |     |

GET article/____mapping

Explicit mapping
```
PUT my_index_name
{
    "mappings": {
        "properties": {
            "my_field_name1": {
                "type": "my_field_type1"
            },
            "my_field_name2": {
                "type": "my_field_type2"
            }
        }
    }
```

We can send a `PUT` request to the `_mapping` endpoint for an index and include our new field names and types under the `properties` field

```
PUT my_index_name/_mapping
{
  "properties": {
            "my_field_name1": {
                "type": "my_field_type1"
            },
            "my_field_name2": {
                "type": "my_field_type2"
            }
        }
}
```

|              |                                                                                                                                                                   |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `boolean`    | Represents `true` and `false` values                                                                                                                              |
| `long`       | A floating point number                                                                                                                                           |
| `double`     | A signed 64-bit integer                                                                                                                                           |
| `keyword`    | A field that contains a single keyword value and is often used in sorting, aggregations, and term-level queries.                                                  |
| `text`       | A field that contains a string value; it is analyzed before being stored and is often used for full-text search.                                                  |
| `completion` | A field that contains a list of completion suggestions used for auto-complete suggestions.                                                                        |
| `object`     | A JSON object                                                                                                                                                     |
| `nested`     | A field that contains an array of objects. Each object can have its properties and data types and can be queried independently of the other objects in the array. |
| `geospatial` | A field that contains a geospatial data type such as a point, line, or polygon.                                                                                   |
| `ip`         | A field that contains an IP address.                                                                                                                              |
| `date`       | A field that contains a date value. It is indexed as a numerical field.                                                                                           |

Text field/ 
Tokenizer, lowercasing, removing stop words.

keyword field 
Save as it is

### Script/ runtime fields

- `lang`: This specifies the language the script is written in. The default is Painless.
    
- `source`: This contains the script to be executed or the ID for the stored script.
    
- `params`: This specifies any named parameters passed into the script as variables. We use parameters instead of hard-coded values to decrease the compile time.
```
  "script": {
    "lang":   "...",
    "source": "...",
    "params": { ... }
  }
```

Adding the runtime field. Will calculate only when searching, can save storage.
```
PUT product/_mapping
{
  "runtime": {
    "price_plus_tax": {
      "type": "double",
      "script": {
        "source": "emit(doc['price'].value * 1.5)"
      }
    }
  }
}
```

Analyzer:

1 Tokenization, 2 Normalization, 
3 Stemming(Change word to original form), Althorithmic stemming(words are reduced to their base form based on a set of rules), Dictionary stemming(Using customized dictionary for mapping)
Combining algorithmic and dictionary stemming can result in a fast and accurate stemming technique.

Token graph: The primary purpose of a token graph is to improve search relevance by examining the connections between tokens (words) in a query, allowing queries to match various phrases that convey the same idea. A set of words can be regarded the same as another series.

Using the same analyzer when store and search for most of the cases

An analyzer in Elasticsearch comprises three main components:

- Character filtering (operate on the original text)
	There are three types of character filters available in Elasticsearch:
	
	- The HTML strip filter
		Could customize html strip with escaped tags
```
		PUT htmlstrip_test
		{
		  "settings": {
		    "analysis": {
		      "analyzer": {
		        "removes_htmltags_analyzer": {
		          "type": "custom", 
		          "tokenizer": "keyword",
		          "char_filter": ["htmlstrip_without_ptag"]
		        }
		      },
		      "char_filter":{
		        "htmlstrip_without_ptag":{
		          "type":"html_strip",
		          "escaped_tags": [
		            "p"
		          ]
		        }
		      }
		    }
		  }
		}`

```
	- The mapping character filter
```
		Get /_analyze
		{
		  "tokenizer": "keyword",
		  "char_filter": [
		    {
		      "type": "mapping",
		      "mappings": [
		        "btw => by the way",
		        "omw => on my way",
		        "idk => i don't know",
		        "imho => in my humble opinion",
		        "lmk => let me know"
		      ]
		    }
		  ],
		  "text": "btw, I'm omw to the store"
		}
	    
```
	- The pattern-replace filter
```
		Get /_analyze
		{
		  "tokenizer": "keyword",
		  "char_filter": [
		    {
		      "type": "pattern_replace",
		      "pattern": "[A-Z]",
		      "replacement": " "
		    }
		  ],
		  "text": "The Quick fOx is rUnninG"
		}
```
- Tokenization (tokenize)
	-- Standard tokenizer
    
	- Whitespace tokenizer
	    
	- Keyword tokenizer
	    
	- Character group tokenizer
		    - `tokenize_on_chars`: This is the defined character list based on which the tokenizer splits the text.
			- `max_token_length`: This is the maximum token length, which defaults to 255.
	- The n-gram tokenizer
		    The **n-gram tokenizer** is a tokenizer that generates a list of nn-grams from the input text.
		    elastic: ["ela", "las", "ast", "sti", "tic"]
			- `min_gram`: This is the minimum length of characters in a gram. The default value is 1.
			- `max_gram`: This is the maximum length of characters in a gram. The default value is 2.
			- `token_chars`: They are character classes that should be included in a token, such as: `letter`, `digit`, `whitespace`, etc. Moreover, the default list `[]` can be used to keep all characters.
	- Edge n-gram tokenizer
		    The **edge n-gram** tokenizer only generates nn-grams containing a word’s beginning (prefixes). For example, if the text is `"elastic"`, then the edge nn-gram will produce the following:
			["e", "el", "ela", "elas", "elasti", "elastic"]
```
				PUT edge_ngram_test
				{
				  "settings": {
				    "analysis": {
				      "analyzer": {
				        "my_edge_ngram_analyzer": {
				          "type": "custom", 
				          "tokenizer": "my_edge_ngram_tokenizer"
				        }
				      },
				      "tokenizer":{
				        "my_edge_ngram_tokenizer":{
				          "type": "edge_ngram",
				          "min_gram": 3,
				          "max_gram": 7
				        }
				      }
				    }
				  }
				}
```
	- Path hierarchy
	The path hierarchy tokenizer splits the text based on a hierarchical value like a filesystem path. For example, the user/elasticsearch/bin string will be tokenized to the following:"user/", "user/elasticsearch", "user/elasticsearch/bin"

- Token filter (operate on the tokens)

	-can be broadly categorized into three types:
	
	- Normalization filters
		- The `lowercase` token filter for converting text to lowercase.
		- The `asciifolding` token filter for converting non-Latin characters to their ASCII equivalents.
		- The `keep_types` token filter to retain only alphabetic characters.
		
	- Stemming filters
		还原词形态
```
	    GET /_analyze
		{
		  "tokenizer": "standard",
		  "filter":["stemmer", 
		            {
		            "type": "stemmer",
		            "language": "spanish"
		            }
		          ],
		  "text": "running means correr in spanish"
		}
```
	- Miscellaneous filters
	**Miscellaneous filters** are a category of token filters that do not fit into the more common categories, such as normalization or stemming filters. These filters are typically used for specialized purposes or to provide additional functionality beyond what is available through other filters.
		- The `stop` token filter for removing stop words.
    
		- The `unique` token filter for removing the duplication tokens.
	    
		- The `synonym` token filter to generate synonym words for `"running"` and `"street"`, i.e., `"racing"` and `"road"`.
-
```
PUT /my_index_name
{
  "setting": {
    "analysis": {
      "analyzer": {
        "my_analyzer_name": {
          "type": "custom",
          "char_filter": [specify a list of char filters],
          "tokenizer": specify a tokenizer,
          "filter": [specify a list of token filters]
        }
      }
    }
  }
}

```

- **Stop token filter:** It removes stop words from a token stream. By default, the filter removes the following English stop words: `a`, `an`, `and`, `are`, `as`, `at`, `be`, `but`, `by`, `for`, `if`, `in`, `into`, `is`, `it`, `no`, `not`, `of`, `on`, `or`, `such`, `that`, `the`, `their`, `then`, `there`, `these`, `they`, `this`, `to`, `was`, `will`, and `with`.

- **Synonym token filter:** This is a useful tool for managing synonyms in the analysis process. Synonyms—words with similar or identical meanings (e.g., “buy” and “purchase”)—can be specified in a configuration file, making it easy to handle them. Another option is to use the `synonym_graph` token filter, which can handle multi-word synonyms accurately by constructing a token graph.

- **Unique token filter:** It removes duplicate tokens from a stream. For example, it converts the text `"running in the street street"` to `"running in the street"`. It can be useful in situations where we want to ensure that a field only contains unique values or reduce the size of an index by eliminating duplicate tokens.

- **Truncate token filter:** It truncates the tokens that exceed a specific character length. For example, if the character length is `6` then the token `"graduation"` will be truncated to `"gradua"`. The character length defaults to `10`.

- **Length token filter:** It removes tokens shorter or longer than the specified character lengths.

- **n-gram token filter:** The nn-gram token filter creates a stream of overlapping nn-gram characters from the original tokens. The nn-grams are contiguous sequences of nn characters from a given text. For example, if the text is `"hello"` and nn is equal to `2`, then the 2-gram token filter will break down the text into the following tokens: `["he", "el", "ll", "lo"]`. The nn-grams token filter generates all possible nn-grams from each token in the input stream and adds them to the token stream. We can also use the edge nn-grams token filter, which only outputs nn-grams that start at the beginning of a token (prefix).

- **Predicate script token filter:** It is designed to filter tokens based on the condition defined in a Painless script. The script evaluates each token in the input stream and determines if it meets the specified condition. If a token meets the condition, it is included in the output stream. Otherwise, it is filtered out.

Could define different analyzer for different fields.

```
"mappings": { 
	"properties": 
		{ "title": 
			{ "type": "text", "analyzer": "english_analyzer" }, 
			
	"description": 
		{ "type": "text", "analyzer": "ik_analyzer" } 
	} 
}
```

Build-in tokenizer

- Standard analyzer
- Whitespace analyzer
- Keyword analyzer
- Fingerprint analyzer
- Language analyzer
```

PUT my_index_name
{
  "mappings": {
      "properties": {
        "my_field": {
          "type": "text",
          "analyzer": "standard"
        }
    }
  }
}
```
#### Defining the built-in analyzer with additional parameters


```
PUT testing_builtin_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "custom_standard_analyzer": { 
          "type":  "standard",
          "stopwords": ["the", "is"]
        }
      }
    }
  }
}

```