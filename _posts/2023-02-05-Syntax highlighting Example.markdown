---
layout: post
title:  "Syntax highlighting 색상 커스텀"
date:   2023-02-05
category: dev
tags: [jekyll]
---

Syntax highlighting 색상 커스텀을 했다.


java:
```java
import javax.sql.DataSource;
import java.sql.Connection;

@SpringBootTest
@Slf4j
class DBTests {

    @Qualifier("db1DataSource")
    @Setter(onMethod_ = {@Autowired})
    private DataSource db1DataSource;

    @Qualifier("db2DataSource")
    @Setter(onMethod_ = {@Autowired})
    private DataSource db2DataSource;

    @Test
    public void testDataSourceConnection() {
        try {
            Connection con1 = db1DataSource.getConnection();
            Connection con2 = db2DataSource.getConnection();
            log.info("" + con1);
            log.info("" + con2);
        } catch (Exception e) {
            fail(e.getMessage());
        }
    }
}
```

<br /><br />sql:
```sql
select tmp.bno, title, content, regdate, updatedate
from tb_blog as b
         join (select bno
               from tb_blog
               where 1 = 1
                   and title like concat('%', 'zz', '%')
                  or content like concat('%', 'zz', '%')
               order by bno desc
               limit 10 offset 10) as tmp
              on tmp.bno = b.bno;
```

<br /><br />js:
```javascript
function sayHello(name) {
  if (!name) {
    console.log('Hello World');
  } else {
    console.log(`Hello ${name}`);
  }  
}  

function myFunc(a, b) {
    return a * b;
}
document.getElementById('demo').innerHTML = myFunc(4, 3);
```

<br /><br />python:
```python
def func():
     # function body
     print("hello world!")

     def setup(app):
         # enable Pygments json lexer
         try:
             import pygments
             if pygments.__version__ >= '1.5':
                 # use JSON lexer included in recent versions of Pygments
                 from pygments.lexers import JsonLexer
             else:
                 # use JSON lexer from pygments-json if installed
                 from pygson.json_lexer import JSONLexer as JsonLexer
         except ImportError:
             pass  # not fatal if we have old (or no) Pygments and no pygments-json
         else:
             app.add_lexer('json', JsonLexer())

         return {"parallel_read_safe": True}

words = ['cat', 'window', 'defenestrate']
for w in words:
   print w, len(w)
```
