we can use the `GET` HTTP method and the `_cluster/health` API to retrieve information about cluster health.

`Get _cluster/health` will be translated to `curl -XGET "http://localhost:9200/_cluster/health"`.

Creating index:
`PUT name`

Create document in an index
`POST/PUT my_index_Name/_doc` 
`{`
  `"my_field_name": "my_value"`
`}`

To add the dataset to the `products` index and perform various tasks, you can utilize the following `_bulk` request. This `_bulk` request will create 50 documents, representing different products, in the `products` index.

```
POST /products/_bulk
{ "index": { "_id": "1" } }
{ "name": "iPhone 13", "description": "The latest smartphone from Apple. Features a powerful A15 Bionic chip and a stunning Super Retina XDR display.", "price": 999, "category": "Electronics", "brand": "Apple", "color": "Midnight Blue", "availability": "In Stock", "rating": 4.5, "seller": "Apple Store" }
{ "index": { "_id": "2" } }
{ "name": "Samsung Galaxy S21", "description": "A high-end Android smartphone with a vibrant Dynamic AMOLED display and an impressive camera system.", "price": 899, "category": "Electronics", "brand": "Samsung", "color": "Phantom Black", "availability": "In Stock", "rating": 4.2, "seller": "Samsung Store" }
{ "index": { "_id": "3" } }
{ "name": "Sony WH-1000XM4", "description": "Wireless noise-canceling headphones with industry-leading noise cancellation and immersive sound.", "price": 349, "category": "Electronics", "brand": "Sony", "color": "Silver", "availability": "Out of Stock", "rating": 4.7, "seller": "Audio World" }
```
#### Indexing a document with a specified ID[](https://www.educative.io/courses/elasticsearch-fundamentals-indexing-and-querying-data/crud-operations#Indexing-a-document-with-a-specified-ID)

If our document contains a unique identifier, such as a `user_account` field or some other value, it is recommended to use it as an ID for the document.

`PUT my_index_Name/_doc/{_id}` 
`{`
  `"my_field_name": "my_value"`
`}`

Using the `_create` endpoint in Elasticsearch can prevent an existing document from being overwritten when we index a document with an ID that already exists. Typically, when we use the `_doc` endpoint to index a document with an existing one, the previous document is replaced by the new one.

`PUT meio/_create/1`
`{`
  `"name": "xiaomeiou"`
`}`

We can send a `GET` request to the `_doc` endpoint by specifying the document `_id` as a param.

`GET index_name/_doc/{_id}`

We can send a `POST` request to the `_update` endpoint by specifying the `_id` as params and providing fields to update in the request body.

`POST meio/_update/1`
`{` 
  `"doc":{`
      `"name": "meiop"`
  `}`
`}`

We can send a `DELETE` request to the `_doc` endpoint and specify the document `_id` as params in the request.

`DELETE index_name/_doc/{_id}`

