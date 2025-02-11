---
layout: post
title: "[ELK] ElasticSearch의 인덱스 템플릿(Template) 설정하기 "
subtitle: "mapping, setting 및 alias 정보를 template으로 설정 후 인덱스에 자동으로 적용하기 / lifecycle policy 추가 및 템플릿 적용"    
comments: true
categories : ELK
date: 2021-03-27
background: '/img/posts/mac.png'
---

이번 글에서는 Elasticsearch에서 인덱스 템플릿을 사용하는 이유와 
사용 방법에 대해서 자세히 다룰 예정이다.   

- - - 

## 1. Index Template   

`인덱스 템플릿은 말 그대로 인덱스 생성을 위한 템플릿을 미리 설정해 놓고 해당 템플릿을 이용해 
인덱스를 생성하는 것을 말한다.`    

`Json 형태의 document를 인덱싱할 때, 매핑 정보를 정해주지 않으면 
엘라스틱 서치가 자동으로 인덱스 매핑을 해주는데, 이를 dynamic mapping(동적 매핑)이라고 한다.`      

아래와 같이 인덱스에 매핑된 값을 확인할 수 있다.   

```
GET index-name/_mapping
```

`이러한 동적 매핑은 편리하기도 하지만, 의도하지 않는 타입이 매핑이 되거나 
숫자 타입의 경우 범위가 가장 넓은 long으로 매핑되는 등 불필요한 용량을 
차지하여 성능에 영향을 미칠 수도 있다.`      

따라서 템플릿을 설정해서 인덱스에 맵핑된 필드 타입을 최적화하여 미리 정의해주면, 
    엘라스틱 서치의 성능 튜닝에도 도움이 된다.    

또한, `템플릿을 사용하는 이유 중 하나는 여러 인덱스들을 패턴으로 매칭하여 설정한 필드 정보 및 settings 들을 
자동으로 적용해 줄 수 있다.`         
아래와 같이 일별로 생성되는 인덱스가 있다고 가정해보면, 
    템플릿을 하나 설정해 놓으면, 해당 템플릿에 설정 해놓은 필드 mappings 및 settings 정보들을 
    동일하게 적용할 수 있다.   


```
// 아래는 'summary*' 패턴으로 생성된 인덱스의 예이다.   

summary-20230503   
summary-20230504   
summary-20230505   
```

`위와 같이 인덱스를 생성할 때 설정해놓은 템플릿의 mapping 및 settings 정보를 
이용하여 인덱스를 생성할 수 있는 기능이 바로 template이다.`     

참고로 [dynamic template](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-templates.html)도 
제공하며, 해당 내용은 공식문서를 참고하자.   

또한 `template을 이용하여 생성된 인덱스에 자동으로 alias를 매핑해 줄 수도 있다.`

이제 직접 template을 생성하여, 인덱스가 생성될 때 설정한 mapping, settings 및 alias 정보들이 적용 되는지 확인해보자.      


- - - 

## 2. template 생성하기   

먼저 기본적인 template 요청은 아래와 같다.   

```
// 모든 index의 템플릿을 조회      
GET _template

// 모든 인덱스 cat api를 통해 조회  
GET _cat/templates?v

// 인덱스 template을 조회   
GET _template/my-index

// 인덱스 template을 삭제     
DELETE _template/my-index
```

이제 사용할 인덱스 템플릿을 생성해보자.   

```
PUT _template/summary-template
{
    "index_patterns": ["summary*"],
    "settings": {
        "max_result_window": 50000,
        "index.mapping.total_fields.limit": 10000,
        "number_of_shards": 5
    },
    "mappings" : {
      "_doc" : {
        "properties" : {
          "id" : {
            "type" : "keyword"
          },
          "name" : {
            "type" : "text"
          },
          "quantity" : {
            "type" : "long"
          },
          "createdAt" : {
            "type": "date"
          }
        }
      }
    },
    "aliases": {
      "summary": {}
    }
}
```

`위 템플릿에서 인덱스 패턴을 지정해 주었기 때문에, 아래와 같이 인덱스가 생성될 때 자동으로 지정한 mappings 및 settings 가 
적용된다.`      

> 위 코드에서 lastic search 7.0 부터 _doc 부분을 제거해야 한다.    

> 물론 일별 뿐 아니라 주 단위 월 단위 모두 가능하다.   

```
summary-20230503
summary-20230504
summary-20230505
```

생성된 인덱스를 확인해보면, 우리가 템플릿으로 설정해놓은 mappgins 및 settings 정보들이 적용 된 것을 확인할 수 있다.   

```
GET summary-20230503

GET summary-20230503/_settings
GET summary-20230503/_mapping
```

`또한, aliases 하위에 summary라는 alias를 지정해 주었다.`   
해당 인덱스 패턴에 매칭되는 인덱스가 생성될 때, summary라는 alias와 매핑되는 것을 확인 할 수 있다.   

> alias 이름과 index 이름과 동일하게 지정하면, 에러를 발생시킨다.   

```
GET _cat/aliases?v

alias      index            filter routing.index routing.search
summary    summary-20230503 -      -             -
```

더 자세한 내용은 [공식문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-template.html)를 
참고해보자.  

- - - -

## 3. LifyCycle Policy 생성 및 템플릿에 추가   

인덱스가 daily로 생성이 되어 쌓인다고 가정해보면, 적절한 retention 기간을 정하고 
그에 따라 주기적으로 제거해주어야 한다.   
이를 자동으로 제거해주는 기능이 kibana의 lifecycle 관리 메뉴에서 제공한다.   

<img width="1400" alt="Image" src="https://github.com/user-attachments/assets/ce652e06-26bf-433e-9634-003de4538a4e" />    

`Index Lifecycle Policies 화면에서 Create Policy 버튼을 클릭하여 정책을 생성하자.`   

<img width="1176" alt="Image" src="https://github.com/user-attachments/assets/fb9e6cc0-2775-4fb7-8746-48091113400e" />   

<img width="1168" alt="Image" src="https://github.com/user-attachments/assets/b1d428f7-47dd-4c32-897e-97fafe9ff709" />   

위 그림과 같이 이름을 작성하고, 휴지통 버튼을 클릭하면, Delete Phase를 설정할 수 있다.  
여기서는 30일이 지난 인덱스에 대해서 자동으로 삭제되도록 하였다.   

아래와 같이 생성한 policy를 인덱스에 추가해줄 수 있다.   

```
PUT mylogs-pre-ilm*/_settings
{
  "index": {
    "lifecycle": {
      "name": "summary-policy"
    }
  }
}
```

`위를 템플릿에 추가하여 인덱스가 생성될 때 자동으로 lifecycle이 적용되도록 할 수도 있다.`   

```
PUT /_template/my-log-template
{
  "index_patterns": [
    "my-log-*"
  ],
  "settings": {
    "index": {
      "lifecycle": {
        "name": "test-policy",
      }
    }
  },
  "mappings": {
// ... 
```




- - - 

**Reference**    

<https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html>   
<https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-templates.html>    
<https://www.elastic.co/guide/en/elasticsearch/reference/current/set-up-lifecycle-policy.html>   

{% highlight ruby linenos %}

{% endhighlight %}


{%- if site.disqus.shortname -%}
    {%- include disqus.html -%}
{%- endif -%}

