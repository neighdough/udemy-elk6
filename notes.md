# Connecting to your cluster

# Create a Mapping for MovieLens

* schema definition
```
curl -H "Content-Type:application/json" -XPUT localhost:9200/movies -d'
curl -H "Content-Type: application/json" -XGET localhost:9200/movies/_mapping/movie?pretty
{
  "mappings":{
    "movie":{
      "properties":{
        "year":{"type":"date"}
      }
    }
  }
}'

* field types
"properties":{
  "user_id":{
    "type":"long"
  }
}

* field index
"properties":{
  "genre":{
    "index": "false"
  }
}

* field analyzer
"properties":{
  "description::{
    "analyzer":"english"
  }
}
```
  character filters
  	remove HTML enconding character conversion, etc.
  tokenizer
  	split strings, etc.
  token filter
  	stemming, convert case, synonyms, etc.

#  Hacking CURL
```
  mkdir ~/bin
  cd bin
  vim curl
  # ! /bin/bash
  /usr/bin/curl -H "Content-Type: application/json" "$@"
  chmod a+x curl
```
#  Import a Single Movie
```
curl -XPUT localhost:9200/movies/movie/109487 -d'
{
"genre":["IMAX","Sci-Fi"],
"title":"Interstellar",
"year":2014
}'
```
#  Insert Many Movies at Once
```
curl -XPUT localhost:9200/_bulk -d 
curl -XPUT localhost:9200/_bulk?pretty --data-binary @movies.json
```
#  Updating Data
curl -XPOST localhost:9200/movies/movie/109487/_update -d '
{
  "doc":{
  	"title":"Interstellar"
  }
}'

#  Deleting Data
### search for title
```
curl -XGET localhost:9200/movies/_search?q=Dark
```
delete document using id
```
curl -XDELETE localhost:9200/movies/movie/58559?pretty

# Dealing with Concurrency
* Optimistic conncurrency control
- retry_on_conflicts=<Number of times to retry>
```
curl -XPOST localhost:9200/movies/movie/109847/_update?retry_on_conflict=5 -d '
{
"doc": {
"title": "Interstellar"
}
}'
```
#  Using Analyzers and Tokenizers
If you want exact match only, must use Keyword mapping
Partial matching, use Text Mapping
```
curl -XGET localhost:9200/movies/movie/_search?pretty -d '
{
"query": {
"match": {
"title": "Star Trek"
}
}
}'

curl -XGET localhost:9200/movies/movie/_search?pretty -d '
{
"query": {
"match_phrase": {
"genre": "sci"
}
}
}'
```
Need to disable analyzers if an exact match is needed which requires full deletion of index
```
curl -XDELETE localhost:9200/movies
curl -XPUT localhost:9200/movies -d '
{
"mappings": {
  "movie": {
    "properties": {
      "id": {"type": "integer"},
      "year": {"type": "date"},
      "genre": {"type": "keyword"},
  		"title": {"type": "text", "analyzer": "english"}
    }
  }
}
}'
```
### Reimport data
```
curl -XPUT localhost:9200/_bulk?pretty --data-binary @movies.json

Doesn't return anything because it's now search full string, not partial match
curl -XGET localhost:9200/movies/movie/_search?pretty -d '
{
"query": {
  "match": {
  	"genre": "sci"
  }
}
}'
```
#  Data Modeling
* Parent Child Relationships
```
curl -XPUT localhost:9200/series -d '
{
"mappings": {
  "movie": {
  	"properties": {
  		"film_to_franchise": 
  			{"type": "join",
  				"relations": {"franchise": "film"}
  			}
  		}
  	}
  }
}'
```
Search for all children of Star Wars
```
curl -XGET localhost:9200/series/movie/_search?pretty -d '
{
"query": {
  "has_parent": 
  	{"parent_type": "franchise", 
  		"query": 
  		{"match": 
  			{"title":"Star Wars"}
  		}
  	}
  }
}'
```
Search for parent of The Force Awakens
```
curl -XGET localhost:9200/series/movie/_search?pretty -d '
{
"query": {
  "has_child": 
  	{"type": "film", 
  		"query": 
  		{"match": 
  			{"title":"The Force Awakens"}
  		}
  	}
  }
}'
```
#  Using Query-String Search
query lite - look for URI Search on Elastic site
/movies/movie/_search?q=title:star
/movies/movie/_search?q=+year:>2010+title:trek
Search strings need to be URL encoded
/movies/movie/_search?q=%2Byear$....
%2B -> + 
Fragile, and prone to security issues. Best for one-off tests

#  Using JSON Search 
- queries: return data in terms of relevance
- filters: more efficient and can be cached, used to ask y/n question in data
  gte: Greater than Equal
```
curl -XGET localhost:9200/movies/movie/_saerch?pretty -d '
{
"query":{
  "bool":{
  	"must":{"term":{"title":"trek"}},
  	"filter":{"range":{"year":{"gte":2010}}}
  }
}
}'
```
* Types of Filters
- term
- terms
- range
- exists
- missing
- bool (must, must_not, should (OR))

* Types of Queries
- match_all (implied, default query)
- match
- multi_match
- bool

* Syntax
- queries wrapped in "query":{} block
- filters wrapped in "filter":{} block

#  Full-text vs phrase Search
- slop value: order matters, but don't care if words are adjacent to each other
  "match_phrase":{"title":{"query":"star beyond", "slop": 1}
  Star Trek Beyond, Star Wars Beyond
- proximity query
  "slop": 100, accepts up to 100 terms in between values

# Pagination
return results in blocks, 0 index
"from":2
"size":0

# Sorting
..._search?sort=year&pretty
- Strings are complicated if they are stored as field analyzer
- One workaround is to store the field twice if you need to sort by text
  raw field is used for sorting
```
  {"mappings":{
  	"movie":{
  		"properties":{
  			"title":{
  				"type":"text",
  				"fields":{
  					"raw":{
  						"type":"keyword",
  						}
  					}
  				}
  			}
  		}
  	}
```
.../_search?sort=title.raw
# Using Filters
```
"query":{"bool":
  {"must":{"match":{"genre":"Sci-Fi"}},
   "must_not": {"match": {"title":"trek"}},
   "filter:{"range":{"year":{"gte":2010,"lt":2015}}}
  }
```
# Fuzzy Queries
- AUTO no fuzziness tolerance
- 0 for 1-2 strings
- 1 for 3-5 character strings
- 2 for anything else
```
{"query":
  {"fuzzy":
  	{"title":{"value": "intrsteller", "fuzziness":2}}
  }
}
```
# Partial matching
```
{"query":
  {"prefix":
  	{"year": "201"}
  }
}

{"query":
  {"wildcard":
  	{"year": "201*"}
  }
}

{"query":
  {"regexp":
  	{"year": ".*"}
  }
}
```
# N-Grams and Search as you Type
- query-time search
```
{"query":
  "match_phrase_prefix":{
  	"title":{"query":"star trek","slop": 1}
  }
}
```
- index-time with N-grams
"star"
unigram: [s,t,a,r]
bigram: [st,ta,ar]
trigram: [sta, tar]
4-gram: [star]

# Importing Data from Scripts
- Downloaded a custom json make script and load script using elasticsearch lib

# Logstash overview
- can anonymize personal data or exclude completely
- derive structure from unsructured data
- 'stash' -> output types
- geolocation

# Importing Apache Access Logs with Logstash
```
    sudo apt update && sudo apt install logstash
    #  create logstash config file
    sudo vim /etc/logstash/conf.d/logstash.conf
    cd /usr/share/logstash
    # need to modify command in lesson to point to logstash path
    sudo bin/logstash --path.settings /etc/logstash -f /etc/logstash/conf.d/logstash.conf
```
# Import Data from PostgreSQL
- Modified from lesson to use PostgreSQL instead of MySQL

# Importing Data from MySQL using Logstash

## configure logstash

```
sudo vim /etc/logstash/conf.d/mysql.conf
```
```
input{
  jdbc{
    jdbc_connection_string => "jdbc:mysql://localhost:3306/movielens
    jdbc_user => "root"
    jdbc_password => "password"
    jdbc_driver_library => "/path/to/jdbc/driver"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    statement => "select * from movies"
  }
}
output{
  stdout {codec => json_lines}
  elasticsearch{
    "hosts" => "localhost:9200"
    "index" => "movielens-sql"
    "document_type" => "data"
  }
}
```    
```
sudo /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/mysql.conf
```
# Importting Data from AWS S3 using Logstash
```
input {
  s3{
    bucket => "bucket-name"
    access_key_id => "accesskey"
    secret_access_key => "secretaccesskey"
  }
}
```

# Integrating Spark and Hadoop with Elasticsearch
## ******** Compatibility Note ***********
Spark currently only works with Java 8, and since Oracle changed their 
licensing model, Java 8 is no longer available from the apt repo


- Download and install spark
- grab data
- install Spark add on for elastic
    * mvnrepository.com and search for org.elasticsearch
- run spark in interactive shell
```
./spark-2.4.3-bin-hadoop2.7/bin/spark-shell --packages org.elasticsearch:elasticsearch-spark-20_2.11:6.7.2
```
```
scala> import org.elasticsearch.spark.sql._
scala> case class Person(ID:Int, name:String, age:Int, numFriends:Int)
scala> def mapper(line:String): Person = {
  | val fields = line.split(',')
  | val person:Person = Person(fields(0).toInt, fields(1), fields(2).toInt, fields(3).toInt)
  | return person
  | }
scala> import spark.implicits._
scala> val lines = spark.sparkContext.textFile("fakefriends.csv")
scala> val people = lines.map(mapper).toDF()
```

# Buckets and Metrics

# Installing Kibana
```
sudo apt install kibana
sudo vim /etc/kibana/kibana.yml
#uncomment server.host line
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable kibana.service
sudo /bin/systemctl start kibana.service
```
