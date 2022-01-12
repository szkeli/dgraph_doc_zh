# 语言和RDF类型 (Language and RDF types)

`RDF N-Quad`允许为字符串值和`RDF`类型指定一种语言。语言是使用`@lang`指定的。例如：

``` rdf
<0x01> <name> "Adelaide"@en .
<0x01> <name> "Аделаида"@ru .
<0x01> <name> "Adélaïde"@fr .
<0x01> <dgraph.type> "Person" .
```

请参阅[如何在查询中处理语言字符串](https://dgraph.io/docs/query-language/graphql-fundamentals/#language-support)。

`RDF`类型通过标准的`^^`分隔符附加到字面量值上。例如：

``` rdf
<0x01> <age> "32"^^<xs:int> .
<0x01> <birthdate> "1985-06-08"^^<xs:dateTime> .
```

受支持的`RDF`数据类型和存储数据的相应内部类型如下所示：

存储类型|`Dgraph`类型
----|-----
\<xs:string>|string
\<xs:dateTime>|dateTime
\<xs:date>|dateTime
\<xs:int>|int
\<xs:integer>|int
\<xs:boolean>|bool
\<xs:double>|float
\<xs:float>|float
\<geo:geojson>|geo
\<xs:password>|password
\<http://www.w3.org/2001/XMLSchema#string>|string
\<http://www.w3.org/2001/XMLSchema#dateTime>|dateTime
\<http://www.w3.org/2001/XMLSchema#date>|dateTime
\<http://www.w3.org/2001/XMLSchema#int>|int
\<http://www.w3.org/2001/XMLSchema#positiveInteger>|int
\<http://www.w3.org/2001/XMLSchema#integer>|int
\<http://www.w3.org/2001/XMLSchema#boolean>|bool
\<http://www.w3.org/2001/XMLSchema#double>|float
\<http://www.w3.org/2001/XMLSchema#float>|float

请参阅[RDF Schema类型](https://dgraph.io/docs/query-language/schema/#rdf-types)一节，以了解RDF类型如何影响变更(mutation)和存储。