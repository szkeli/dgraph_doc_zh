# Expand 函数

可以使用`expand()`函数在节点外展开谓词。想要使用`expand()`，类型系统是必需的。请参阅关于[类型系统](./type_system.md)的一节，以检查如何设置类型节点。本节的其余部分假设您熟悉该节。

使用`expand`函数有两种方法。

可以将类型传递给`expand()`以展开该类型中的所有谓词。

查询示例：列出`Harry Potter`的电影：

``` dql
{
  all(func: eq(name@en, "Harry Potter")) @filter(type(Series)) {
    name@en
    expand(Series) {
      name@en
      expand(Film)
    }
  }
}
``` 

``` json 
{
  "data": {
    "all": [
      {
        "name@en": [
          "Harry Potter",
          "Harry Potter"
        ],
        "name@fi": "Harry Potter (elokuvasarja)",
        "name@hu": "Harry Potter filmsorozat",
        "name@sl": "Filmska serija Harry Potter",
        "name@da": "Harry Potter-filmserien",
        "name@ca": "Saga de Harry Potter",
        "series.films_in_series": [
          {
            "name@en": [
              "Harry Potter and the Chamber of Secrets",
              "Harry Potter and the Chamber of Secrets"
            ],
            "name@fi": "Harry Potter ja salaisuuksien kammio",
            "name@zh": "哈利·波特与密室",
            "name@pt-BR": "Harry Potter e a Câmara Secreta",
            "name@hu": "Harry Potter és a Titkok Kamrája",
            "name@sl": "Harry Potter in dvorana skrivnosti",
            "name@et": "Harry Potter ja saladuste kamber",
            "name@lv": "Harijs Poters un Noslēpumu kambaris",
            "name@de": "Harry Potter und die Kammer des Schreckens",
            "tagline@en": "Hogwarts is back in session.",
            "netflix_id": "60024925",
            "initial_release_date": "2002-11-03T00:00:00Z",
            "rottentomatoes_id": "harry_potter_and_the_chamber_of_secrets",
            "traileraddict_id": "harry-potter-and-the-chamber-of-secrets",
            "metacritic_id": "harrypotterandthechamberofsecrets"
          },
          {
            "name@en": [
              "Harry Potter and the Deathly Hallows - Part I",
              "Harry Potter and the Deathly Hallows - Part I"
            ],
            "name@fi": "Harry Potter ja kuoleman varjelukset, osa 1",
            "name@zh": "哈利·波特与死亡圣器（上）",
            "name@pt-BR": "Harry Potter e as Relíquias da Morte – Parte 1",
            "name@hu": "Harry Potter és a Halál ereklyéi 1.",
            "name@sl": "Harry Potter in Svetinje smrti - 1. del",
            "name@pt-PT": "Harry Potter e os Talismãs da Morte: Parte 1",
            "name@fr": "Harry Potter et les reliques de la mort - 1ère partie",
            "name@ja": "ハリー・ポッターと死の秘宝 PART1",
            "name@ro": "Harry Potter și Talismanele Morții. Partea 1",
            "name@uk": "Гаррі Поттер і смертельні реліквії: частина 1",
            "name@id": "Harry Potter and the Deathly Hallows - Bagian 1",
            "name@iw": "הארי פוטר ואוצרות המוות",
            "name@lt": "Haris Poteris ir Mirties relikvijos. 1 dalis",
            "name@zh-Hant": "哈利波特－死神的聖物1",
            "name@sr": "Хари Потер и реликвије Смрти: Први део",
            "name@el": "Ο Χάρι Πότερ και οι Κλήροι του Θανάτου - Μέρος 1",
            "name@sv": "Harry Potter och dödsrelikerna",
            "name@pl": "Harry Potter i Insygnia Śmierci: Część I",
            "name@ar": "هاري بوتر ومقدسات الموت – الجزء 1",
            "name@bg": "Хари Потър и Даровете на Смъртта: Първа част",
            "name@et": "Harry Potter ja surma vägised: osa 1",
            "name@lv": "Harijs Poters un Nāves dāvesti: Pirmā daļa",
          }
        ]
      }
    ]
  }
}
```

如果`_all_`作为参数传递给`expand()`，则要展开的谓词将是分配给给定节点的类型中的字段的联合。

`_all_`关键字要求节点具有类型。`Dgraph`将查找已分配给节点的所有类型，查询类型以检查它们具有哪些属性，并使用这些属性计算要展开的谓词列表。

例如，考虑一个具有`Animal`和`Pet`类型的节点，它们有以下定义：

``` dql
type Animal {
  name
  species
  dob
}

type Pet {
  owner
  veterinarian
}
```

当在这个节点上调用`expand(_all_)`时，`Dgraph`首先检查节点的类型`Animal`和`Pet`。然后，它将获得`Animal`和`Pet`的定义，并根据它们的类型定义构建谓词列表。

``` dql
name
species
dob
owner
veterinarian
```

> 注意：对于字符串谓词，展开只返回没有标记语言的值(参见语言首选项)。所以通常需要添加`name@fr`或`name@.`以及扩展查询。

## 在过滤中展开

在传出边缘的类型上展开查询支持筛选器。例如，`expand(_all_) @filter(type(Person))`将在所有谓词上展开，但只包含目标节点类型为`Person`的边。因为只有`uid`类型的节点可以具有类型，所以该查询将过滤掉任何标量值。

请注意，`expand`函数目前不支持其他类型的过滤器和指令。过滤器需要使用`type`函数来允许过滤器。支持逻辑与或操作。例如，`expand(_all_) @filter(type(Person) OR type(Animal))`将只扩展指向任一类型节点的边。