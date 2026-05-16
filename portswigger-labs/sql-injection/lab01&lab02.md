## SQL injection UNION attack, determining the number of columns returned by the query & SQL injection UNION attack, finding a column containing text -Burp 复现

## 实验信息

- 平台：PortSwigger Web Security Academy
- 漏洞：SQL injection
- Lab: SQL injection UNION attack, determining the number of columns returned by the query & SQL injection UNION attack, finding a column containing text
- 难度：Practitioner

## 漏洞原理

这两个实验同属于 SQL injection 中的 **UNION attack** 类别。核心原理是：当应用程序将用户输入拼接到 SQL 查询中且未做充分过滤时，攻击者可以通过注入 `UNION SELECT` 语句，将恶意查询结果与原始查询结果合并返回。

**Lab01** — 确定查询返回的列数：在进行 UNION 注入前，必须知道原始查询返回了多少列，因为 UNION 要求前后两个 SELECT 的列数一致。常用的探测方法有两种：
- **ORDER BY 递增法**：依次注入 `' ORDER BY 1--`、`' ORDER BY 2--`……直到返回错误为止，最后一个不报错的序号即为列数。
- **UNION SELECT NULL 法**：`' UNION SELECT NULL--`、`' UNION SELECT NULL,NULL--`……逐列增加 NULL，直到不报错为止。

**Lab02** — 查找包含文本的列：确定列数后，需要找出哪些列的数据类型是字符串（varchar/text），只有字符串类型的列才能承载我们想要提取的数据。通过 `' UNION SELECT 'testString',NULL,NULL...--` 逐列替换 NULL 为测试字符串，观察页面是否正常回显，找出字符串列的位置。

两者的区别在于：Lab01 只需确定列数（用 NULL 即可），Lab02 需要进一步确认哪些列能承载文本数据。

## 测试过程

### Lab01: Determining the number of columns

1. 进入 Lab，页面是一个商品分类过滤器，URL 中包含 `?category=xxx` 参数。在参数值后添加 `'` 测试，确认存在 SQL 注入。

2. 使用 ORDER BY 法确定列数。注入 `' ORDER BY 1--`，页面正常返回，说明至少 1 列。
![order by 1](images/lab01-order-by1.png)

3. 依次递增：注入 `' ORDER BY 2--`，页面正常，说明至少 2 列。
![order by 2](images/lab01-order-by2.png)

4. 继续递增：注入 `' ORDER BY 3--`，页面正常，说明至少 3 列。
![order by 3](images/lab01-order-by3.png)

5. 注入 `' ORDER BY 4--`，页面返回 Internal Server Error，说明原始查询只返回 3 列。
![order by 4](images/lab01-order-by4.png)

6. 使用union select null--法确定列数。null代表列名(字符型)，一个null,页面返回500，说明不是1列
![union select query](images/lab01-query1.png)

7. 依次递增，页面500，说明不是2列
![query with fake category](images/lab01-query2.png)

8. 最终 payload 执行成功，说明是3列，Lab01 完成。
![query success](images/lab01-query3.png)
![lab01 solved](images/lab01-solved.png)

### Lab02: Finding a column containing text

1. Lab02 的入口同样是 category 过滤器。Lab 页面会给出一个随机字符串（Make the database retrieve the string: 'xxxxx'），需要让数据库返回这个字符串。首先使用union select null-- 法确定列数（与 lab01 操作相同），本次为 3 列。
![determine columns](images/lab02-determine-column.png)

2. 从第一列开始逐列测试：`' UNION SELECT 'xxxxx',NULL,NULL--`，观察页面是否正常显示该字符串。
![find position 1](images/lab02-find-pos1.png)

3. 第二列测试：`' UNION SELECT NULL,'xxxxx',NULL--`，确认该列是否为字符串类型。当页面成功显示该字符串时，说明找到了字符串列。
![find position 2](images/lab02-find-pos2.png)

4. 确认字符串列位置后，使用不存在的 category 参数值，让页面回显 UNION 后的结果，成功让数据库返回目标字符串。
![select the string](images/lab02-select.png)

5. Lab solved!
![lab02 solved](images/lab02-sovled.png)

## 利用Payload

**Lab01 核心 Payload：**

```sql
-- 使用 ORDER BY 确定列数
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
' ORDER BY 4--    -- 此处报错，确定列数为3

-- 使用 UNION SELECT 确认
' UNION SELECT NULL,NULL,NULL--

-- 最终攻击 payload（使原始查询无结果，回显 UNION 内容）
' UNION SELECT NULL,NULL,NULL--
```

**Lab02 核心 Payload：**

```sql
-- 已知列数为3，逐一测试哪些列可以存放文本
' UNION SELECT '提供的随机字符串',NULL,NULL--
' UNION SELECT NULL,'提供的随机字符串',NULL--
' UNION SELECT NULL,NULL,'提供的随机字符串'--
```

UNION 注入的两个前提条件：
1. 原始查询和注入查询的**列数必须一致**（通过 ORDER BY 或 NULL 递增法确认）
2. 注入查询的每一列的**数据类型**必须与原始列兼容（用测试字符串逐列探测）

## 个人总结

- 第一，如何利用这个漏洞？

核心思路分两步：**先定列数，再找类型**。用 ORDER BY 或 UNION SELECT NULL 递增直到不报错来确认列数；然后用测试字符串替换每一列的 NULL，观察页面是否正常回显，定位"字符串列"。找到后，将 category 设为无效值使原始查询返回空，页面回显的就是 UNION 后的内容。在后续实验中，会进一步利用这些字符串列提取其他表的数据（如用户名、密码）。

- 第二，为什么会产生这个漏洞？

根本原因是应用使用**字符串拼接**的方式构造 SQL 查询，且未对用户输入做充分过滤或参数化处理。例如 PHP 中 `$sql = "SELECT * FROM products WHERE category = '" . $_GET['category'] . "'"` 这种写法，攻击者的输入会直接嵌入到 SQL 语句中执行。此外，应用将数据库的内部错误（Internal Server Error）直接暴露给了前端，使得攻击者能通过"报错 / 不报错"这种布尔反馈来逐列试探。

- 第三，如何修复这个漏洞？

1. **使用参数化查询（Prepared Statements）**：将 SQL 逻辑与数据分离，用户输入不再被当作 SQL 代码执行，这是最根本的解决方案。
2. **屏蔽数据库错误信息**：生产环境不要将 SQL 错误直接返回给前端，统一返回通用错误页面，增加攻击者的探测成本。
3. **输入校验与白名单**：对于 category 这类参数，优先使用白名单校验（只允许预期的合法值），而非简单黑名单。
