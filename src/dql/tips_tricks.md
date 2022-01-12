# 使用DQL的一些小提示和技巧

## 获取样本数据

使用`has`函数获取一些示例节点：

``` dql
{
  result(func: has(director.film), first: 10) {
    uid
    expand(_all_)
  }
}
```

``` json 
{
  "data": {
    "result": [
      {
        "uid": "0x2b",
        "name@en": "Zehra Yiğit"
      },
      {
        "uid": "0x48",
        "name@en": "Charlton Heston",
        "name@fi": "Charlton Heston",
        "name@zh": "查尔顿·赫斯顿",
        "name@hu": "Charlton Heston",
        "name@sl": "Charlton Heston",
        "name@da": "Charlton Heston",
        "name@ca": "Charlton Heston",
        "name@ko": "찰턴 헤스턴",
        "name@pt": "Charlton Heston",
        "name@no": "Charlton Heston",
        "name@th": "ชาร์ลตัน เฮสตัน",
        "name@cs": "Charlton Heston",
        "name@tr": "Charlton Heston",
        "name@hr": "Charlton Heston",
        "name@es": "Charlton Heston",
        "name@sk": "Charlton Heston",
        "name@nl": "Charlton Heston",
        "name@it": "Charlton Heston",
        "name@fa": "چارلتون هستون",
        "name@vi": "Charlton Heston",
        "name@ru": "Чарлтон Хестон",
        "name@fr": "Charlton Heston",
        "name@ja": "チャールトン・ヘストン",
        "name@ro": "Charlton Heston",
        "name@uk": "Чарлтон Гестон",
        "name@id": "Charlton Heston",
        "name@iw": "צ'רלטון הסטון",
        "name@zh-Hant": "卻爾登·希斯頓",
        "name@sr": "Чарлтон Хестон",
        "name@fil": "Charlton Heston",
        "name@el": "Τσάρλτον Ίστον",
        "name@sv": "Charlton Heston",
        "name@pl": "Charlton Heston",
        "name@ar": "شارلتون هيستون",
        "name@es-419": "Charlton Heston",
        "name@bg": "Чарлтън Хестън",
        "name@lv": "Čārltons Hestons",
        "name@de": "Charlton Heston"
      },
      {
        "uid": "0x4e",
        "name@en": "Rajeev Sharma",
        "name@hi": "राजीव शर्मा"
      },
      {
        "uid": "0x9d",
        "name@en": "Gabriel Nussbaum"
      },
      {
        "uid": "0xb4",
        "name@en": "James Wharton O'Keefe",
        "name@sl": "James Wharton O'Keefe"
      }
      ...
    ]
  }
}
```

## 获取连接点的计数

使用`expand(_all_)`展开节点的边，然后将它们赋值给一个变量。这个变量现在可以用来迭代唯一的邻近节点。然后使用`count(uid)`对一个块中的节点数进行计数：

``` dql
{
  uids(func: has(director.film), first: 1) {
    uid
    expand(_all_) { u as uid }
  }

  result(func: uid(u)) {
    count(uid)
  }
}
```

``` json 
{
  "data": {
    "uids": [
      {
        "uid": "0x2b",
        "director.film": [
          {
            "uid": "0x195896"
          }
        ],
        "name@en": "Zehra Yiğit"
      }
    ],
    "result": [
      {
        "count": 1
      }
    ]
  }
}
```

## 在没有索引的谓词(predicate)上进行搜索

在值变量中使用`has`函数来搜索非索引谓词：

``` dql
{
  var(func: has(festival.date_founded)) {
    p as festival.date_founded
  }
  query(func: eq(val(p), "1961-01-01T00:00:00Z")) {
      uid
      name@en
      name@ru
      name@pl
      festival.date_founded
      festival.focus { name@en }
      festival.individual_festivals { total : count(uid) }
  }
}
```

``` json 
{
  "data": {
    "query": [
      {
        "uid": "0x2d005",
        "name@en": "Kraków Film Festival",
        "name@ru": "Кинофестиваль в Кракове",
        "name@pl": "Krakowski Festiwal Filmowy",
        "festival.date_founded": "1961-01-01T00:00:00Z",
        "festival.focus": [
          {
            "name@en": "Short Film"
          }
        ],
        "festival.individual_festivals": [
          {
            "total": 55
          }
        ]
      }
    ]
  }
}
```

## 根据嵌套的节点值对边进行排序

`Dgraph`排序是基于单个层次的子图。要按更深层次的值对层次排序，可以使用查询变量将嵌套的值提升到要排序的边缘层次。

例如：将史蒂芬·斯皮尔伯格电影中的所有演员按字母顺序排序。演员的名字不能从主演边缘的一次遍历中访问;该名称可以通过`performance.actor`访问：

``` dql 
{
  spielbergMovies as var(func: allofterms(name@en, "steven spielberg")) {
    name@en
    director.film (orderasc: name@en, first: 1) {
      starring {
        performance.actor {
          ActorName as name@en
        }
        # Stars is a uid-to-value map mapping
        # starring edges to performance.actor names
        Stars as min(val(ActorName))
      }
    }
  }

  movies(func: uid(spielbergMovies)) @cascade {
    name@en
    director.film (orderasc: name@en, first: 1) {
      name@en
      starring (orderasc: val(Stars)) {
        performance.actor {
          name@en
        }
      }
    }
  }
}
```

``` json 
{
  "data": {
    "movies": [
      {
        "name@en": "Steven Spielberg",
        "director.film": [
          {
            "name@en": "1941",
            "starring": [
              {
                "performance.actor": [
                  {
                    "name@en": "Akio Mitamura"
                  }
                ]
              },
              {
                "performance.actor": [
                  {
                    "name@en": "Andy Tennant"
                  }
                ]
              },
              {
                "performance.actor": [
                  {
                    "name@en": "Antoinette Molinari"
                  }
                ]
              }
              ...
            ]
          }
        ]
      }
    ]
  }
}
```

## 使用变量获得唯一的结果

为了获得唯一的结果，将节点的边赋值给一个变量。现在可以使用该变量迭代唯一的节点。

例子：从史蒂芬·斯皮尔伯格导演的所有电影中获取所有独特的类型：

``` dql
{
  var(func: eq(name@en, "Steven Spielberg")) {
    director.film {
      genres as genre
    }
  }

  q(func: uid(genres)) {
    name@.
  }
}
```

``` json 
{
  "data": {
    "q": [
      {
        "name@.": "War film"
      },
      {
        "name@.": "Horror"
      },
      {
        "name@.": "Crime"
      },
      {
        "name@.": "Childhood Drama"
      }
      ...
    ]
  }
}
```

## 使用 `checkpwd` 检查密码

将`checkpwd`的结果存储在一个查询变量中，然后将其与1 (checkpwd为true)或0 (checkpwd为false)进行匹配：

``` dql
{  
  exampleData(func: has(email)) {
    uid
    email
    check as checkpwd(pass, "1bdfhJHb!fd")
  }
  userMatched(func: eq(val(check), 1)) {
    uid
    email
  }
  userIncorrect(func: eq(val(check), 0)) {
    uid
    email
  }
}
```

``` json 
{
  "data": {
    "exampleData": [
      {
        "uid": "0x1",
        "email": "tory@email.xyz",
        "checkpwd(pass)": true
      },
      {
        "uid": "0x2711",
        "email": "bill@email.xyz",
        "checkpwd(pass)": false
      },
      {
        "uid": "0x4e21",
        "email": "charity@email.xyz",
        "checkpwd(pass)": false
      },
      {
        "uid": "0x7531",
        "email": "phill@email.xyz",
        "checkpwd(pass)": false
      },
      {
        "uid": "0xc55e",
        "email": "reservations@hotelgriffon.com",
        "checkpwd(pass)": false
      },
      {
        "uid": "0x11303",
        "email": "sfdowntown@hiusa.org",
        "checkpwd(pass)": false
      }
    ],
    "userMatched": [
      {
        "uid": "0x1",
        "email": "tory@email.xyz"
      }
    ],
    "userIncorrect": [
      {
        "uid": "0x2711",
        "email": "bill@email.xyz"
      },
      {
        "uid": "0x4e21",
        "email": "charity@email.xyz"
      },
      {
        "uid": "0x7531",
        "email": "phill@email.xyz"
      }
    ]
  }
}
```

