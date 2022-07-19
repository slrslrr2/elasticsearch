Elasticsearch - 검색엔진, Data 저장소, Restful 검색엔진
Logstash - 파이프라인, Data수집, 데이터 전송
Kibana - Data 시각화, 프론트엔드

| ELK      | RDBMS    |
| -------- | -------- |
| index    | database |
| type     | table    |
| document | row      |
| field    | column   |
| mapping  | schema   |

# 2-1 인덱스 샤드 모니터링

## 인덱스와 샤드

- 프라이머리 샤드(Primary Shard)와 복제본(Replica)
- 인덱스의 settings 설정에서 샤드 갯수 지정
- _cat/shards API를 이용한 샤드 상태 조회

## 모니터링 도구를 이용한 클러스터 모니터링

- Kibana의 모니터링 도구 실행 및 확인
- _cluster/settings API를 이용한 모니터링 실행/중지

> https://esbook.kimjmin.net/03-cluster/3.2-index-and-shards

---



#### 인덱스 생성 

```json
// Request
PUT /books
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}'

// Response
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "books"
}
```

- 한번 만들어진 index는 같은 이름으로 생성 불가능하다.

- number_of_shards는 index 생성 후 변경X

- number_of_replicas 는 중간에 변경 O

  - ```json
    PUT books/_settings
    {
        "number_of_replicas": 0
    }
    ```



#### 구성확인

```json
// Request
GET _cat/nodes?v&h=ip,name,node.role

// Response
ip        name   node.role
127.0.0.1 node-1 cdfhilmrstw
```

- &h : header



#### 모니터링 셋팅 끄기

```json
// Request
GET _cluster/settings

// Response
{
  "persistent" : {
    "xpack" : {
      "monitoring" : {
        "collection" : {
          "enabled" : "true"
        }
      }
    }
  },
  "transient" : { }
}
```

```json
// Request
PUT _cluster/settings
{
  "persistent" : {
    "xpack" : {
      "monitoring" : {
        "collection" : {
          "enabled" : "false"
        }
      }
    }
  }
}

// Response
{
  "acknowledged" : true,
  "persistent" : {
    "xpack" : {
      "monitoring" : {
        "collection" : {
          "enabled" : "false" // "true": 재실행, null 완전삭제
        }
      }
    }
  },
  "transient" : { }
}
```



-----

# 2-2 도큐먼트 CRUDS : 입력, 조회, 수정, 삭제, 검색

#### REST API로 도큐먼트 접근

- 입력|조회|삭제 : PUT|GET|DELETE {\_index}/_doc/{\_id}
- 업데이트 : POST {\_index}/\_update/{\_id}
- 벌크 명령 : POST \_bulk

#### _search API로 도큐먼트 검색

- URI 검색 : ?q=...쿼리...
- 데이터 본문 검색 : {"query" : {...쿼리...}}

> https://esbook.kimjmin.net/04-data/4.1-rest-api

----

bulk로 데이터 넣기

```json
POST _bulk
{"index":{"_index":"test", "_id":"1"}}
{"field":"value one"}
{"index":{"_index":"test", "_id":"2"}}
{"field":"value two"}
{"delete":{"_index":"test", "_id":"2"}}
{"create":{"_index":"test", "_id":"3"}}
{"field":"value three"}
{"update":{"_index":"test", "_id":"1"}}
{"doc":{"field":"value two"}}
{"index":{"_index":"test", "_id":"2"}}
{"field":"value two"}
```



인덱스를 조회하는것

```java
GET test/_doc/1
```

```json
{
  "_index" : "test",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "_seq_no" : 4,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "field" : "value two"
  }
}
```



two라는 값이 어느 인덱스이 있는지 확인

```json
GET test/_search?q=two

GET test/_search?q=field:two
// 이렇게 직접 field명을 지정할 수도 있다.

GET test/_search?q=field:two AND field:value
// AND 조건 넣기
```

```json
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.53899646,
    "hits" : [
      {
        "_index" : "test",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.53899646,
        "_source" : {
          "field" : "value two"
        }
      },
      {
        "_index" : "test",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.53899646,
        "_source" : {
          "field" : "value two"
        }
      }
    ]
  }
}
```



하지만, 위처럼 조회하는것보단 Body에 데이터를 담아 조회한다.

----

# 2-3-1 Query DSL : 풀텍스트

#### 풀텍스트(Full Text) 쿼리

- match
- Match_phrase
- Query_string

#### Relevancy (연관성, 정확도)

- TF / IDF

> https://esbook.kimjmin.net/05-search

-----

```json
POST my_index/_bulk
{"index":{"_id":1}}
{"message":"The quick brown fox"}
{"index":{"_id":2}}
{"message":"The quick brown fox jumps over the lazy dog"}
{"index":{"_id":3}}
{"message":"The quick brown fox jumps over the quick dog"}
{"index":{"_id":4}}
{"message":"Brown fox brown dog"}
{"index":{"_id":5}}
{"message":"Lazy jumping dog"}

GET my_index/_doc/1

GET my_index/_search
GET my_index/_search
{
  "query": {
    "match": {
      "message": "dog"
    }
  }
}

// quick OR dog
GET my_index/_search
{
  "query": {
    "match": {
      "message": "quick dog"
    }
  }
}

// quick AND doc 따로든 같이든 있는것
GET my_index/_search
{
  "query": {
    "match": {
      "message": {
        "query": "quick dog",
        "operator": "and"
      }
    }
  }
}

// "quick dog" 이 구문을 가지고오고싶은경우
GET my_index/_search
{
  "query": {
    "match_phrase": {
      "message": "quick dog"
    }
  }
}


// quick dog 사이에 허용하는 단여 갯수 
// quick dog, quick fast dog
GET my_index/_search
{
  "query": {
    "match_phrase": {
      "message": {
        "query": "quick dog",
        "slop": 1
      }
    }
  }
}
```



## 5.3 정확도 - Relevancy

사용자가 입력한 검색어와 가장 연관성 있는지를 계산하여 score 점수 체크하는 방법

- TF (Term Frequency)
- IDF (Inverse Document Frequency)
- Field Length

> https://esbook.kimjmin.net/05-search/5.3-relevancy

-------



# Query DSL : Bool, Range

#### 복합 (Bool) 쿼리

- https://esbook.kimjmin.net/05-search/5.2-bool

  - **must** : 쿼리가 참인 도큐먼트들을 검색합니다. 

  - **must_not** : 쿼리가 거짓인 도큐먼트들을 검색합니다. 

  - **should** : 검색 결과 중 이 쿼리에 해당하는 **도큐먼트의 점수를 높입니다. **
    - 여기에 들어간 값은 **있어도되고 없어도 되는데**, 있으면 score을 높인다.

  - **filter** : 쿼리가 참인 도큐먼트를 검색하지만 스코어를 계산하지 않습니다. must 보다 검색 속도가 빠르고 캐싱이 가능합니다.
    - 여기에 있는 **값은 있어야**하지만, **점수에는 영향 X**

#### 쿼리에 따른 검색 스코어의 변화

#### 범위(Range) 쿼리

- 숫자값 / 날짜의 범위 검색

----

#### 복합(Bool)쿼리 예제

```json
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "message": {
              "query": "fox" // 필수데이터, 있으면 점수 높임
            }
          }
        },
        {
          "match": {
            "message": "quick" // 필수데이터, 있으면 점수 높임
          }
        }
      ],
      "should": [
        {
          "match": {
            "message": {
              "query": "lazy" // lazy 없어도됨, 있으면 점수 높임
            }
          }
        }
      ],
      "filter": [
        {
          "match" :{
            "message":"lazy" // lazy 무조건 있어야함, 점수 영향X
          }
        }
      ]
    }
  }
}
```

![image-20220717170013047](image-20220717170013047.png)



```json
GET phones/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 400,
        "lte": 800
      }
    }
  }
}

// range 여러개 하고싶을땐 bool must 사용
GET phones/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "price": {
              "gte": 400,
              "lte": 700
            }
          }
        },
        {
          "range": {
            "date": {
              "gte": "2010-01-01",
              "lte": "2015-01-01"
            }
          }
        }
      ]
    }
  }
}
```

-----

# 2-4 데이터 색인과 텍스트 분석

#### 역 인덱스 (Inverted index)

#### 텍스트 분석 과정 (Text Analysis)

- Tokenizer
- Token Filter
- _analyze API 를 이용한 분석 테스트

#### 사용자 정의 analyzer

#### 텍스트 필드에 analyzer 적용

#### _termvectors API를 이용한 term 확인

> https://esbook.kimjmin.net/06-text-analysis/6.1-indexing-data



 ![image-20220717173248087](image-20220717173248087.png)

Elasticsearch는 문자열 필드가 저장될 때 데이터에서 검색어 토큰을 저장하기 위해 여러 단계의 처리 과정을 거칩니다. 이 전체 과정을 **텍스트 분석(Text Analysis)** 이라고 하고 이 과정을 처리하는 기능을 **애널라이저(Analyzer)** 라고 합니다. 

#### 1. 캐릭터 필터

텍스트 데이터가 입력되면 **가장 먼저** 필요에 따라 **전체 문장에서 특정 문자를 대치하거나 제거**하는데 이 과정을 담당



#### 2. 토크나이저

 다음으로는 문장에 속한 단어들을 **텀 단위**로 하나씩 **분리 해 내는 처리 과정**을 거치는데 이 과정을 담당하는 기능입니다.<br>토크나이저는 **반드시 1개**만 적용이 가능합니다. <br>다음은 **`whitespace`** 토크나이저를 이용해서 공백을 기준으로 텀 들을 분리 한 결과입니다.

![image-20220717173639452](image-20220717173639452.png)



#### 3. 토큰 필터

분리된 **텀** 들을 **하나씩 가공**하는 과정을 거칩니다.

![image-20220717173737983](image-20220717173737983.png)

> 1. 여기서는 먼저 <b> `lowercase`</b> 토큰 필터를 이용해서 **대문자를 모두 소문자로** 바꿔줍니다. <br>이렇게 하면 대소문자 구별 없이 검색이 가능하게 됩니다. 대소문자가 일치하게 되어 같은 텀이 된 토큰들은 모두 하나로 병합이 됩니다.

![image-20220717174050341](image-20220717174050341.png)



![image-20220717174219888](image-20220717174219888-8047345.png)

> 2.  <b>`stop`</b> 옵션을 통하여 **불용어를 제거**시킵니다
>
> 텀 중에는 검색어로서의 가치가 없는 단어들이 있는데 이런 단어를 **불용어(stopword)** 라고 합니다. 보통 **a, an, are, at, be, but, by, do, for, i, no, the, to …** 등의 단어들은 불용어로 간주되어 검색어 토큰에서 제외됩니다. `stop`토큰 필터를 적용하면 우리가 만드는 역 인덱스에서 **the**가 제거됩니다.



`snowball`을 통해 형태소 분석 과정을 거쳐서 문법상 변형된 단어를 일반적으로 검색에 쓰이는 기본 형태로 변환해줍니다.<br>

![image-20220717174503863](image-20220717174503863.png)



<b>`snowball`</b>을 통해 필요에 따라서는 **동의어**를 추가 해 주기도 합니다. 

![image-20220717174619170](image-20220717174619170.png)



### 사용자 정의 analyzer 

예제 1

```json
PUT my_index2
{
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "analyzer": "snowball" // mappings 정의
      }
    }
  }
}

// my_index에 id 에 값 생성
PUT my_index2/_doc/1
{
  "message": "The quick brown fox jumps over the lazy dog" 
}

GET my_index2/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "message": "jump"
          }
          
        }
      ]
    }
  }
}
```



![image-20220717182601733](image-20220717182601733.png)

![image-20220717183228087](image-20220717183228087.png)



------

# 2-5 노리(nori) 한글 형태소 분석기

#### Token Filter 추가 설명

#### Nori 설치

- Elasticsearch-plugin install analysis-nori
- _analyze API 를 이용한 한글 분석 테스트

#### Nori 설정

- user_dictionary_ruls를 이용한 사용자 사전 입력
- decompound_mode : 합성어 저장 설정
- Stoptags : Part Of Speech 품사 태그 지정
- nori_readingform : 한자어 번역

-----

> https://esbook.kimjmin.net/06-text-analysis/6.7-stemming/6.7.2-nori


-------

| 구분1                     | 구분2          | game/gameserver | subject | content |
| ------------------------- | -------------- | --------------- | ------- | ------- |
| **char_filter**           | html_strip     | -               | -       | O       |
|                           | mapping        | -               | -       | △       |
|                           | 정규식         | -               | -       | -       |
| **tokenizer**             | standard       |                 |         |         |
|                           | Letter         | -               | -       | -       |
|                           | Whitespace     | -               | -       | -       |
|                           | UAX URL Email  | -               | -       | -       |
|                           | Pattern        | -               | -       | -       |
|                           | path_hierarchy | -               | -       | -       |
| **filter** (token_filter) | Lowercase      | O               | O       | O       |
|                           | uppercase      | -               | -       | -       |
|                           | synonyms_path  | O               | -       | -       |
|                           | **ngram**      | -               | O?      | O?      |
|                           | **Edge NGram** | O               | -       | -       |
|                           | unique         | -               | O       | O       |

