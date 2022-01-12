# 值变量上的数学操作

值变量可以用数学函数组合在一起。例如，这可以用于关联一个分数，然后该分数用于排序或执行其他操作，例如可能用于构建新闻提要、简单推荐系统等。

`Math`语句必须包含在`Math(<exp>)`中，并且必须存储到值变量中。

支持的操作符如下：

操作|接受的类型|作用
---|---|---
`+` `-` `*` `/` `%`| int float| 进行相应的运算
`min` `max` | 除了`geo` `bool`|选择最大或最小值
`<` `>` `<=` `>=` `==` `!=`|除了`geo` `bool`| 返回 `true`或者`false`
`floor` `ceil` `ln` `exp` `sqrt`| `int` `float` |进行相应的运算
`since`| `dateTime`|返回从指定时间开始的浮点数秒数
`pow(a, b)`|`int` `float`|返回a的b次方
`logbase(a, b)`| `int` `float` |将`log(a)`返回到基数`b`
`cond(a, b, c)`|第一个操作数必须是`bool`值|如果a为真，选择b，否则选c

> 如果发生整数溢出，或将操作数传递给数学操作(如`ln`、`logbase`、`sqrt`、`pow`)导致非法操作，`Dgraph`将返回错误。

查询示例：将史蒂芬·斯皮尔伯格的每一部电影的演员数量、类型数量和国家数量相加，得出分数。按得分递减的顺序列出这类电影的前五名：

``` graphql 
{
	var(func:allofterms(name@en, "steven spielberg")) {
		films as director.film {
			p as count(starring)
			q as count(genre)
			r as count(country)
			score as math(p + q + r)
		}
	}

	TopMovies(func: uid(films), orderdesc: val(score), first: 5){
		name@en
		val(score)
	}
}
```

``` json 
{
  "data": {
    "TopMovies": [
      {
        "name@en": "Lincoln",
        "val(score)": 179
      },
      {
        "name@en": "Minority Report",
        "val(score)": 156
      },
      {
        "name@en": "Schindler's List",
        "val(score)": 145
      },
      {
        "name@en": "The Terminal",
        "val(score)": 118
      },
      {
        "name@en": "Saving Private Ryan",
        "val(score)": 99
      }
    ]
  }
}
```

可以在过滤器中使用值变量及其聚合。

查询示例：为每一部史蒂文·斯皮尔伯格`Steven Spielberg`的电影计算一个分数，并带有上映日期的条件，以惩罚10年以上的电影，过滤结果分数：

``` graphql
{
  var(func:allofterms(name@en, "steven spielberg")) {
    films as director.film {
      p as count(starring)
      q as count(genre)
      date as initial_release_date
      years as math(since(date)/(365*24*60*60))
      score as math(cond(years > 10, 0, ln(p)+q-ln(years)))
    }
  }

  TopMovies(func: uid(films), orderdesc: val(score)) @filter(gt(val(score), 2)){
    name@en
    val(score)
    val(date)
  }
}
```

``` json 
{
  "data": {
    "TopMovies": [
      {
        "name@en": "Lincoln",
        "val(score)": 7.927418,
        "val(date)": "2012-10-08T00:00:00Z"
      }
    ]
  }
}
```

通过数学操作计算的值存储在值变量中，因此可以聚合。

查询示例：为`Steven Spielberg`的每一部电影计算一个分数，然后将分数相加：

``` graphql
{
	steven as var(func:eq(name@en, "Steven Spielberg")) @filter(has(director.film)) {
		director.film {
			p as count(starring)
			q as count(genre)
			r as count(country)
			score as math(p + q + r)
		}
		directorScore as sum(val(score))
	}

	score(func: uid(steven)){
		name@en
		val(directorScore)
	}
}
```

``` json 
{
  "data": {
    "score": [
      {
        "name@en": "Steven Spielberg",
        "val(directorScore)": 1865
      }
    ]
  }
}
```