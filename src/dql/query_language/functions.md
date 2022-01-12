# 函数
函数允许基于节点或变量的属性进行筛选。函数可以应用于根查询或过滤器中。
> Dgraph v1.2.0添加了对非索引谓词上的过滤器的支持。

根查询(又名`func:`)中的比较函数(`eq`, `ge`, `gt`, `le`, `lt`)只能应用于索引谓词。从`v1.2`开始，现在可以在`@filter`指令上使用比较函数，甚至可以在没有索引的谓词上使用比较函数。对于大型数据集，在非索引谓词上进行筛选可能会很慢，因为它们需要在使用筛选器的级别上迭代所有可能的值。

根查询或筛选器中的所有其他函数只能应用于索引谓词。

对于字符串值谓词上的函数，如果没有给出语言首选项，则该函数将应用于所有没有语言标记的语言和字符串；如果给定了语言首选项，则该函数只应用于给定语言的字符串。

## 词匹配
### allofterms
解析示例：```allofterms(predicate, "空 格 分开 的词 列表")```

Schema类型：`string`

索引要求：`term`

以任意顺序匹配具有所有指定项的字符串；不区分大小写。

#### 用在根查询中
查询示例：所有名称中包含词条`indiana`和`jones`的节点，以英文返回英文名称和流派：
``` graphql
{
  me(func: allofterms(name@en, "jones indiana")) {
    name@en
    genre {
      name@en
    }
  }
}
```
这将返回：
``` json 
{
  "data": {
    "me": [
      {
        "name@en": "The Adventures of Young Indiana Jones: Passion for Life",
        "genre": [
          {
            "name@en": "History"
          }
        ]
      },
      {
        "name@en": "The Young Indiana Jones Chronicles 2",
        "genre": [
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action Film"
          }
        ]
      },
      {
        "name@en": "The Adventures of Young Indiana Jones: Travels with Father",
        "genre": [
          {
            "name@en": "Family"
          },
          {
            "name@en": "Adventure Film"
          }
        ]
      }
      ...
    ]
  }
}
```

#### 用在过滤器中
查询示例：史蒂芬·斯皮尔伯格所有包含印第安纳和琼斯字样的电影。`@filter(has(director.film))`删除了名字为`Steven Spielberg`但不是导演的节点————数据还包含了一部名为`Steven Spielberg`的电影中的一个角色。
``` graphql
{
  me(func: eq(name@en, "Steven Spielberg")) @filter(has(director.film)) {
    name@en
    director.film @filter(allofterms(name@en, "jones indiana"))  {
      name@en
    }
  }
}
```
将返回：
``` json 
{
  "data": {
    "me": [
      {
        "name@en": "Steven Spielberg",
        "director.film": [
          {
            "name@en": "Indiana Jones and the Temple of Doom"
          },
          {
            "name@en": "Indiana Jones and the Raiders of the Lost Ark"
          },
          {
            "name@en": "Indiana Jones and the Kingdom of the Crystal Skull"
          },
          {
            "name@en": "Indiana Jones and the Last Crusade"
          }
        ]
      }
    ]
  }
}
```


### anyofterms
解析示例：```anyofterms(predicate, "space-separated term list")```
Schema类型：`string`
索引要求：`term`

以任意顺序匹配具有任何指定术语的字符串；不区分大小写。

#### 用在根查询中
查询示例:所有名称包含`poison`或`peacock`的节点。许多返回的节点都是电影，但像`Joan Peacock`这样的人也满足搜索条件，因为没有级联指令，查询就不需要类型：
```graphql
{
  me(func:anyofterms(name@en, "poison peacock")) {
    name@en
    genre {
      name@en
    }
  }
}
```
将返回：
``` json 
{
  "data": {
    "me": [
      {
        "name@en": "The Peacock Spring",
        "genre": [
          {
            "name@en": "Drama"
          }
        ]
      },
      {
        "name@en": "David Peacock"
      },
      {
        "name@en": "Harry Peacock"
      },
      {
        "name@en": "Poison Apple",
        "genre": [
          {
            "name@en": "Family"
          },
          {
            "name@en": "Fantasy"
          },
          {
            "name@en": "Short Film"
          },
          {
            "name@en": "Backstage Musical"
          }
        ]
      }
      ...
    ]
  }
}
```

#### 用在过滤器中
查询示例：所有史蒂文·斯皮尔伯格的战争或间谍电影。`@filter(has(director.film))`删除了名字为`Steven Spielberg`但不是导演的节点————数据还包含了一部名为`Steven Spielberg`的电影中的一个角色：
``` graphql
{
  me(func: eq(name@en, "Steven Spielberg")) @filter(has(director.film)) {
    name@en
    director.film @filter(anyofterms(name@en, "war spies"))  {
      name@en
    }
  }
}
```

将返回：
``` json 
{
  "data": {
    "me": [
      {
        "name@en": "Steven Spielberg",
        "director.film": [
          {
            "name@en": "War Horse"
          },
          {
            "name@en": "War of the Worlds"
          },
          {
            "name@en": "Bridge of Spies"
          }
        ]
      }
    ]
  }
}
```


### 正则表达式
解析示例：```regexp(predicate, /regular-expression/)``` 或者不区分大小写```regexp(predicate, /regular-expression/i)```
Schema类型：`string`
索引要求：`trigram`

通过正则表达式匹配字符串。正则表达式语言是`go`正则表达式的语言。

查询示例：在根节点，在名称开头匹配Steven Sp，后面跟着任意字符。对于每个这样匹配的uid，匹配包含ryan的影片。注意allofterms的不同之处，它只匹配ryan，但是正则表达式搜索也会在terms内匹配，比如bryan：

``` graphql
{
  directors(func: regexp(name@en, /^Steven Sp.*$/)) {
    name@en
    director.film @filter(regexp(name@en, /ryan/i)) {
      name@en
    }
  }
}
```
将返回：
``` json 
{
  "data": {
    "directors": [
      {
        "name@en": "Steven Spencer"
      },
      {
        "name@en": "Steven Spielberg",
        "director.film": [
          {
            "name@en": "Saving Private Ryan"
          }
        ]
      },
      {
        "name@en": "Steven Spielberg And The Return To Film School"
      },
      {
        "name@en": "Steven Spohn"
      },
      {
        "name@en": "Steven Spieldal"
      },
      {
        "name@en": "Steven Spielberg"
      },
      {
        "name@en": "Steven Sperling"
      },
      {
        "name@en": "Steven Spencer"
      },
      {
        "name@en": "Steven Spielberg"
      },
      {
        "name@en": "Steven Spurrier"
      },
      {
        "name@en": "Steven Spielberg"
      },
      {
        "name@en": "Steven Spurrier"
      }
    ]
  }
}
```

**技术细节：**
`Trigram`是由三个连续的符文组成的子串。例如，Dgraph有三元组合Dgr, gra, rap, aph。
为了保证正则表达式匹配的效率，Dgraph使用了三元组索引。也就是说，Dgraph将正则表达式转换为trigram查询，使用trigram索引和trigram查询来查找可能的匹配，并仅对可能的匹配项应用完整的正则表达式搜索。

#### Dgraph中正则表达式的高效使用和限制

** TODO **


### 模糊匹配
解析规则：```match(predicate, string, distance)```
Schema类型：`string`
索引类型：`trigram`

通过计算到字符串的[Levenshtein](https://en.wikipedia.org/wiki/Levenshtein_distance)距离来匹配谓词值，也称为模糊匹配。距离参数必须大于零(0)。使用更大的距离值可以产生更多但不太精确的结果。

查询示例：在根查询处，模糊匹配节点类似于`Stephen`，距离值小于等于8：
``` graphql
{
  directors(func: match(name@en, Stephen, 8)) {
    name@en
  }
}
```
将返回：

``` json 
{
  "data": {
    "directors": [
      {
        "name@en": "Stephen Chen"
      },
      {
        "name@en": "Step Up 3D"
      },
      {
        "name@en": "Steven Bauer"
      },
      {
        "name@en": "Stephen Chow"
      },
      {
        "name@en": "Joseph"
      },
      {
        "name@en": "Ron Steinman"
      },
      {
        "name@en": "Steven Blum"
      },
      {
        "name@en": "The Chef"
      },
    ]
  }
}
```

### 全文搜索

解析规则：```alloftext(predicate, "space-separated text")```或者```anyoftext(predicate, "space-separated text")```
Schema类型：`string`
索引要求：`fulltext`

应用带有词干分析和停止词的全文本搜索，以查找与所有或任何给定文本匹配的字符串。

以下步骤在索引生成和处理全文搜索参数时应用：
1. 标记化(根据Unicode单词边界)。
2. 转换为小写的。
3. unicode规范化(到[正规化形式KC](https://unicode.org/reports/tr15/#Norm_Forms))。
4. 词干分析使用特定于语言的词干分析器(如果语言支持的话)。
5. 停止单词删除(如果语言支持)。

`Dgraph`使用[bleve](https://github.com/blevesearch/bleve)进行全文搜索索引。请参阅bleve语言特定的[停止单词列表](https://github.com/blevesearch/bleve/tree/master/analysis/lang)。

下表包含所有支持的语言，相应的国家代码，词干和停止词过滤支持：
> todo


查询示例：所有包含`dog`, `dogs`, `bark`, `bark`, `bark`等的名称。停止字删除消除和`which`：

``` graphql
{
  movie(func:alloftext(name@en, "the dog which barks")) {
    name@en
  }
}
```
将返回：
``` json 
{
  "data": {
    "movie": [
      {
        "name@en": "Black Dogs Barking"
      },
      {
        "name@en": "Elliott Erwitt: 'I Bark at Dogs"
      },
      {
        "name@en": "Barking Dogs"
      },
      {
        "name@en": "Barking Dogs Never Bite"
      },
      {
        "name@en": "Do You Hear the Dogs Barking?"
      },
      {
        "name@en": "But The Word Dog Doesn’t Bark"
      }
      ...
    ]
  }
}
```

### 等于

解析示例：
* `eq(predicate, value)`
* `eq(val(varName), value)`
* `eq(predicate, val(varName))`
* `eq(count(predicate), value)`
* `eq(predicate, [val1, val2, val3])`
* `eq(predicate, [$var1, "value2", ..., $varN])`

Schema类型：`int`, `float`, `bool`, `string`, `dateTime`
索引要求：在查询根处使用`eq(predicate, ...)`形式时需要索引(见下表)。对于查询根的`count`(谓词)，需要`@count`索引。对于变量，值是作为查询的一部分计算的，因此不需要索引：

类型 | 索引要求
----| -----
int | int
float | float
bool | bool
string | exact hash term fulltext
dateTime| dateTime
布尔常量分别为`true`和`false`，因此使用`eq`时，就变成了，例如，`eq(boolPred, true)`。

查询示例：查询具有十三种流派的电影，并返回十三种流派的名字：
``` graphql
{
  me(func: eq(count(genre), 13)) {
    name@en
    genre {
      name@en
    }
  }
}
```
将返回：
``` json 
{
  "data": {
    "me": [
      {
        "name@en": "Stay Tuned",
        "genre": [
          {
            "name@en": "Horror"
          },
          {
            "name@en": "Science Fiction"
          },
          {
            "name@en": "Thriller"
          },
          {
            "name@en": "Family"
          }
          ...
        ]
      },
       {
        "name@en": "Trollhunters",
        "genre": [
          {
            "name@en": "Horror"
          },
        ]
       }
       ...
    ]
  }
}
```

查询示例：导演叫史蒂文，他曾经导演过1、2或3部电影：
``` graphql
{
  steve as var(func: allofterms(name@en, "Steven")) {
    films as count(director.film)
  }
  stevens(func: uid(steve)) @filter(eq(val(films), [1,2,3])) {
    name@en
    numFilms: val(films)
  }
}
```
将返回：
``` json 
{
  "data": {
    "stevens": [
      {
        "name@en": "Steven Wilsey",
        "numFilms": 1
      },
      {
        "name@en": "Steven Bratter",
        "numFilms": 1
      },
      {
        "name@en": "Steven Saussey",
        "numFilms": 1
      },
      {
        "name@en": "Steven Ray Morris",
        "numFilms": 1
      }
      ...
    ]
  }
}
```

### 小于、小于或等于、大于、大于或等于
解析示例：(假设不等号是`IE`)
* IE(predicate, value)
* IE(val(varName), value)
* IE(predicate, val(varName))
* IE(count(predicate), value)

`IE`的可能取值：
* `le`: 小于或等于(less than or equal to)
* `lt`: 小于(less than)
* `ge`: 大于或等于(greater than or equal to)
* `gt`: 大于(greater than)

Schema类型：`int`, `float`, `string`, `dateTime`

索引要求：当不等式查询`IE`用在根查询时，需要下表的索引要求。对于用在根查询的`count(predicate)`，则需要`@count`索引。对于变量，因为变量是在查询中计算的一部分，所以不需要索引。
类型|索引要求
---| ---
int|int
float| float
string|string
dateTime|dateTime

查询示例：列举`Ridley Scott`在1980年之前导演的电影：
``` graphql
{
  me(func: eq(name@en, "Ridley Scott")) {
    name@en
    director.film @filter(lt(initial_release_date, "1980-01-01"))  {
      initial_release_date
      name@en
    }
  }
}
```
将返回：
``` json
{
  "data": {
    "me": [
      {
        "name@en": "Ridley Scott",
        "director.film": [
          {
            "initial_release_date": "1979-05-25T00:00:00Z",
            "name@en": "Alien"
          },
          {
            "initial_release_date": "1977-12-01T00:00:00Z",
            "name@en": "The Duellists"
          }
        ]
      },
      {
        "name@en": "Ridley Scott"
      }
    ]
  }
}
```

查询示例：以史蒂文为导演的同名电影，曾执导过100多名演员。

``` graphql
{
  ID as var(func: allofterms(name@en, "Steven")) {
    director.film {
      num_actors as count(starring)
    }
    total as sum(val(num_actors))
  }

  dirs(func: uid(ID)) @filter(gt(val(total), 100)) {
    name@en
    total_actors : val(total)
  }
}
```
将返回：
``` json 
{
  "data": {
    "dirs": [
      {
        "name@en": "Steven Conrad",
        "total_actors": 122
      },
      {
        "name@en": "Steven Knight",
        "total_actors": 102
      },
      {
        "name@en": "Steven Zaillian",
        "total_actors": 123
      },
      {
        "name@en": "Steven Spielberg",
        "total_actors": 1665
      },
      {
        "name@en": "Steven Brill",
        "total_actors": 417
      }
      ...
    ]
  }
}
```

查询示例：每类电影超过30000部的电影。因为在类型上没有指定顺序，所以顺序将通过UID。`count`索引记录节点的边数，并使这样的查询更多：

``` graphql
{
  genre(func: gt(count(~genre), 30000)){
    name@en
    ~genre (first:1) {
      name@en
    }
  }
}
```
将返回：
``` json 
{
  "data": {
    "genre": [
      {
        "name@en": "Documentary film",
        "~genre": [
          {
            "name@en": "Wild Things"
          }
        ]
      },
      {
        "name@en": "Drama",
        "~genre": [
          {
            "name@en": "José María y María José: Una pareja de hoy"
          }
        ]
      },
      {
        "name@en": "Comedy",
        "~genre": [
          {
            "name@en": "José María y María José: Una pareja de hoy"
          }
        ]
      },
      {
        "name@en": "Short Film",
        "~genre": [
          {
            "name@en": "Side Effects"
          }
        ]
      }
    ]
  }
}
```

查询示例：叫`Steven`的导演和他们的电影`initial_release_date`大于电影`Minority Report`的电影：

``` graphql
{
  var(func: eq(name@en,"Minority Report")) {
    d as initial_release_date
  }
  me(func: eq(name@en, "Steven Spielberg")) {
    name@en
    director.film @filter(ge(initial_release_date, val(d))) {
      initial_release_date
      name@en
    }
  }
}
```

将返回：
``` json 
{
  "data": {
    "me": [
      {
        "name@en": "Steven Spielberg",
        "director.film": [
          {
            "initial_release_date": "2012-10-08T00:00:00Z",
            "name@en": "Lincoln"
          },
          {
            "initial_release_date": "2011-12-04T00:00:00Z",
            "name@en": "War Horse"
          },
          {
            "initial_release_date": "2005-06-13T00:00:00Z",
            "name@en": "War of the Worlds"
          },
          {
            "initial_release_date": "2002-06-17T00:00:00Z",
            "name@en": "Minority Report"
          }
          ...
        ]
      },
      {
        "name@en": "Steven Spielberg"
      },
      {
        "name@en": "Steven Spielberg"
      },
      {
        "name@en": "Steven Spielberg"
      }
    ]
  }
}
```

### between
解析规则：`between(predicate, startDateValue, endDateValue)`
Schema类型：`Scalar Type`, `dateTime`, `int`, `float`, `string`
索引要求：`dateTime`, `int`, `float`, `exact`

返回与索引值的包含范围相匹配的节点。`between`关键字对索引执行范围检查，以提高查询效率，有助于防止对大数据集进行大范围查询时运行缓慢。

`between`关键字的一个常见用例是在由`dateTime`索引的数据集中进行搜索。下面的示例查询演示了这个用例。

查询示例：1977年首次上映的电影，按类型列出：

``` graphql
{
  me(func: between(initial_release_date, "1977-01-01", "1977-12-31")) {
    name@en
    genre {
      name@en
    }
  }
}
```
将返回：
``` json 
{
  "data": {
    "me": [
      {
        "name@en": "SST: Death Flight",
        "genre": [
          {
            "name@en": "Drama"
          }
        ]
      },
      {
        "name@en": "Lucio Flavio",
        "genre": [
          {
            "name@en": "Crime Fiction"
          },
          {
            "name@en": "Drama"
          }
        ]
      },
      {
        "name@en": "Uyarnthavargal",
        "genre": [
          {
            "name@en": "World cinema"
          },
          {
            "name@en": "Drama"
          },
          {
            "name@en": "Musical Drama"
          },
          {
            "name@en": "Backstage Musical"
          },
          {
            "name@en": "Tamil cinema"
          }
        ]
      }
      ...
    ]
  }
}
```

### uid
解析示例：
* `q(func:uid(<uid>))`
* `predicate @filter(uid(<uid1>, ..., <uidn>))`
* `predicate @filter(uid(a))` 对变量`a`使用
* `q(func:uid(a, b))` 对变量`a`和`b`使用
* `q(func:uid($uids))` 对一组uid使用，例如`[0x1, 0x2, 0x3, ..., 0xn]`

仅将当前查询级别的节点过滤为给定`uid`集合中的节点。

对于查询变量`a`, `uid(a)`表示存储在`a`中的`uid`集合，对于值变量`b`, `uid(b)`表示`uid`到`value`映射的`uid`集合。对于两个或多个变量，`uid(a,b，…)`表示所有变量的并集。

`uid(<uid>)`与标识函数一样，即使节点没有任何边，也将返回所请求的`uid`。

查询示例：按已知`UID`查询`Priyanka Chopra`的胶片：

``` graphql
{
  films(func: uid(0x2c964)) {
    name@hi
    actor.film {
      performance.film {
        name@hi
      }
    }
  }
}
```
将返回：

``` json 
{
  "data": {
    "films": [
      {
        "name@hi": "प्रियंका चोपड़ा",
        "actor.film": [
          {
            "performance.film": [
              {
                "name@hi": "यकीन"
              }
            ]
          },
          {
            "performance.film": [
              {
                "name@hi": "सलाम-ए-इश्क़"
              }
            ]
          }
          ...
        ]
      }
    ]
  }
}
```

查询示例：塔拉吉·汉森的电影按类型分类：

``` graphql
{
  var(func: allofterms(name@en, "Taraji Henson")) {
    actor.film {
      F as performance.film {
        G as genre
      }
    }
  }

  Taraji_films_by_genre(func: uid(G)) {
    genre_name : name@en
    films : ~genre @filter(uid(F)) {
      film_name : name@en
    }
  }
}
```
将返回：

``` json 
{
  "data": {
    "Taraji_films_by_genre": [
      {
        "genre_name": "War film",
        "films": [
          {
            "film_name": "Talk to Me"
          }
        ]
      },
      {
        "genre_name": "Horror",
        "films": [
          {
            "film_name": "Satan's School for Girls"
          }
        ]
      },
      {
        "genre_name": "Indie film",
        "films": [
          {
            "film_name": "Once Fallen"
          },
          {
            "film_name": "Peep World"
          },
          {
            "film_name": "Hustle & Flow"
          },
          {
            "film_name": "All or Nothing"
          },
          {
            "film_name": "Hair Show"
          }
        ]
      },
      {
        "genre_name": "Crime",
        "films": [
          {
            "film_name": "Term Life"
          }
        ]
      }
      ...
    ]
  }
}
```

查询示例：`Taraji Henson`的电影按体裁数量排序，体裁列表按`Taraji`在每个体裁中制作的电影数量排序：

``` graphql
{
  var(func: allofterms(name@en, "Taraji Henson")) {
    actor.film {
      F as performance.film {
        G as count(genre)
        genre {
          C as count(~genre @filter(uid(F)))
        }
      }
    }
  }
  Taraji_films_by_genre_count(func: uid(G), orderdesc: val(G)) {
    film_name : name@en
    genres : genre (orderdesc: val(C)) {
      genre_name : name@en
    }
  }
}

```
将返回：
``` json 
{
  "data": {
    "Taraji_films_by_genre_count": [
      {
        "film_name": "Date Night",
        "genres": [
          {
            "genre_name": "Comedy"
          },
          {
            "genre_name": "Crime Fiction"
          },
          {
            "genre_name": "Romance Film"
          },
          {
            "genre_name": "Thriller"
          },
          {
            "genre_name": "Action Film"
          },
          {
            "genre_name": "Romantic comedy"
          },
          {
            "genre_name": "Action/Adventure"
          },
          {
            "genre_name": "Action Comedy"
          },
          {
            "genre_name": "Chase Movie"
          },
          {
            "genre_name": "Screwball comedy"
          }
        ]
      },
      {
        "film_name": "I Can Do Bad All by Myself",
        "genres": [
          {
            "genre_name": "Drama"
          },
          {
            "genre_name": "Comedy"
          },
          {
            "genre_name": "Romance Film"
          },
          {
            "genre_name": "Romantic comedy"
          },
          {
            "genre_name": "Comedy-drama"
          },
          {
            "genre_name": "Musical Drama"
          },
          {
            "genre_name": "Backstage Musical"
          },
          {
            "genre_name": "Musical comedy"
          }
        ]
      }
      ...
    ]
  }
}
```

### uid_in
解析示例：
* `q(func: ...) @filter(uid_in(predicate, <uid>))`
* `predicate1 @filter(uid_in(predicate2, <uid>))`
* `predicate1 @filter(uid_in(predicate2, [<uid1>, ..., <uid2>]))`
* `predicate1 @filter(uid_in(predicate2, uid(myVariable)))`
Schema类型：UID
索引要求：无

虽然`uid`函数基于`uid`在当前级别上过滤节点，但`uid_in`函数允许沿着边缘向前查找，以检查它是否指向特定的`uid`。这通常可以节省额外的查询块，并避免返回边缘。

`uid_in`不能在根查询下使用。它接受多个`UID`作为它的参数，并接受一个`UID`变量(它可以包含`UID`的映射)。

查询示例：`Marc Caro`和`Jean-Pierre Jeunet`的协作关系`(UID 0x99706)`。如果`Jean-Pierre Jeunet`的`UID`是已知的，那么以这种方式查询就不需要用一个块将他的`UID`提取到一个变量中，也不需要对`~director.film`进行额外的边遍历和过滤器：

``` graphql
{
  caro(func: eq(name@en, "Marc Caro")) {
    name@en
    director.film @filter(uid_in(~director.film, 0x99706)) {
      name@en
    }
  }
}
```
将返回：
``` json 
{
  "data": {
    "caro": [
      {
        "name@en": "Marc Caro"
      }
    ]
  }
}
```

如果不知道`Jean-Pierre Jeunet`的`UID`，还可以查询他的`UID`，并在`UID`变量中使用它：

``` graphql
{
  getJeunet as q(func: eq(name@fr, "Jean-Pierre Jeunet"))

  caro(func: eq(name@en, "Marc Caro")) {
    name@en
    director.film @filter(uid_in(~director.film, uid(getJeunet) )) {
      name@en
    }
  }
}
```
将返回：
``` json 
{
  "data": {
    "q": [],
    "caro": [
      {
        "name@en": "Marc Caro",
        "director.film": [
          {
            "name@en": "The City of Lost Children"
          },
          {
            "name@en": "Delicatessen"
          },
          {
            "name@en": "The Bunker of the Last Gunshots"
          },
          {
            "name@en": "L'évasion"
          }
        ]
      }
    ]
  }
}
```

### has
解析规则：`has(predicate)`

Schema使用类型：所有

确定节点是否具有特定谓词。

查询示例：前五位导演和他们所有的电影都有一个上映日期记录。导演已经指导了至少一个电影————等效语义`gt(count(director.film), 0)`：

``` graphql
{
  me(func: has(director.film), first: 5) {
    name@en
    director.film @filter(has(initial_release_date))  {
      initial_release_date
      name@en
    }
  }
}
```


将返回：
``` json
{
  "data": {
    "me": [
      {
        "name@en": "Zehra Yiğit"
      },
      {
        "name@en": "Charlton Heston",
        "director.film": [
          {
            "initial_release_date": "1982-09-23T00:00:00Z",
            "name@en": "Mother Lode"
          },
          {
            "initial_release_date": "1972-03-02T00:00:00Z",
            "name@en": "Antony and Cleopatra"
          },
          {
            "initial_release_date": "1988-12-21T00:00:00Z",
            "name@en": "A Man for All Seasons"
          }
        ]
      },
      {
        "name@en": "Rajeev Sharma",
        "director.film": [
          {
            "initial_release_date": "2012-01-01T00:00:00Z",
            "name@en": "Nabar"
          },
          {
            "initial_release_date": "2014-05-30T00:00:00Z",
            "name@en": "47 to 84"
          }
        ]
      }
      ...
    ]
  }
}
```

### Geolocation 地理位置

#### 变更
要使用geo函数，您需要谓词上的索引。
``` schema
loc: geo @index(geo) .
```

下面是如何添加一个点：
``` dql 
{
  set {
    <_:0xeb1dde9c> <loc> "{'type':'Point','coordinates':[-122.4220186,37.772318]}"^^<geo:geojson> .
    <_:0xeb1dde9c> <name> "Hamon Tower" .
    <_:0xeb1dde9c> <dgraph.type> "Location" .
  }
}
```

下面是如何将一个多边形与一个节点关联。添加多多边形也是类似的：
``` dql
{
  set {
    <_:0xf76c276b> <loc> "{'type':'Polygon','coordinates':[[[-122.409869,37.7785442],[-122.4097444,37.7786443],[-122.4097544,37.7786521],[-122.4096334,37.7787494],[-122.4096233,37.7787416],[-122.4094004,37.7789207],[-122.4095818,37.7790617],[-122.4097883,37.7792189],[-122.4102599,37.7788413],[-122.409869,37.7785442]],[[-122.4097357,37.7787848],[-122.4098499,37.778693],[-122.4099025,37.7787339],[-122.4097882,37.7788257],[-122.4097357,37.7787848]]]}"^^<geo:geojson> .
    <_:0xf76c276b> <name> "Best Western Americana Hotel" .
    <_:0xf76c276b> <dgraph.type> "Location" .
  }
}
```
以上的例子是从我们的[SF旅游数据集](https://github.com/dgraph-io/benchmarks/blob/master/data/sf.tourism.gz?raw=true)中挑选出来的。

#### 查询

**near 附近查询**

解析示例：`near(predicate, [long, lat], distance)`

Schema类型：`geo`

索引要求：`geo`

匹配所有由谓词给出的位置在距离geojson坐标`[long, lat]`米范围内的实体。

查询示例：旧金山金门公园某点1000米(1公里)以内的旅游目的地：

```graphql
{
  tourist(func: near(loc, [-122.469829, 37.771935], 1000) ) {
    name
  }
}
```

将返回：
``` json 
{
  "data": {
    "tourist": [
      {
        "name": "Peace Lantern"
      },
      {
        "name": "De Young Museum"
      },
      {
        "name": "Buddha"
      },
      {
        "name": "Morrison Planetarium"
      },
      {
        "name": "Chinese Pavillion"
      },
      {
        "name": "Strawberry Hill"
      }
      ...
    ]
  }
}
```

**within 在给定范围之内**

**contains 包含**

**intersects 相交**
