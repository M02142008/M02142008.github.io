<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>紫薯公主的 CTF 作战室</title>
<style>
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  body {
    font-family: "Consolas", "Microsoft YaHei", monospace;
    background:
      radial-gradient(circle at top left, rgba(88, 130, 255, 0.18), transparent 35%),
      radial-gradient(circle at bottom right, rgba(179, 80, 255, 0.15), transparent 40%),
      #05070d;
    color: #d6e4ff;
    min-height: 100vh;
    padding: 24px 16px 40px;
    overflow-x: hidden;
  }

  .container {
    max-width: 480px;
    margin: 0 auto;
  }

  /* 顶部状态栏 */
  .status-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px 14px;
    border: 1px solid rgba(120, 180, 255, 0.25);
    border-radius: 12px;
    background: linear-gradient(135deg, rgba(17, 28, 55, 0.85), rgba(10, 12, 30, 0.85));
    box-shadow: 0 0 18px rgba(80, 140, 255, 0.15);
    margin-bottom: 18px;
  }

  .status-bar .label {
    font-size: 12px;
    color: #7dd3fc;
    letter-spacing: 2px;
  }

  .status-bar .value {
    font-size: 13px;
    color: #e0f2fe;
    font-weight: bold;
  }

  /* 主卡片 */
  .card {
    border: 1px solid rgba(120, 180, 255, 0.3);
    border-radius: 18px;
    background: linear-gradient(160deg, rgba(14, 22, 48, 0.92), rgba(9, 11, 28, 0.92));
    box-shadow:
      0 0 25px rgba(56, 130, 255, 0.18),
      inset 0 0 30px rgba(255, 255, 255, 0.03);
    padding: 24px 20px;
    margin-bottom: 16px;
    position: relative;
    overflow: hidden;
  }

  .card::before {
    content: "";
    position: absolute;
    top: -40%;
    right: -20%;
    width: 220px;
    height: 220px;
    background: radial-gradient(circle, rgba(120, 90, 255, 0.25), transparent 70%);
    pointer-events: none;
  }

  .hero-title {
    font-size: 26px;
    line-height: 1.35;
    color: #ffffff;
    margin-bottom: 10px;
    text-shadow: 0 0 12px rgba(125, 211, 252, 0.45);
  }

  .hero-title .line-1 {
    background: linear-gradient(90deg, #7dd3fc, #ffffff);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
  }

  .hero-title .line-2 {
    background: linear-gradient(90deg, #c4b5fd, #f0abfc);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
  }

  .hero-sub {
    font-size: 13px;
    color: #a5b4fc;
    margin-bottom: 18px;
    letter-spacing: 1px;
  }

  .code-box {
    background: rgba(0, 0, 0, 0.35);
    border-left: 3px solid #7dd3fc;
    border-radius: 8px;
    padding: 12px 14px;
    font-size: 13px;
    color: #bae6fd;
    margin-bottom: 14px;
  }

  .code-box .cmd {
    color: #86efac;
  }

  .desc {
    font-size: 13px;
    line-height: 1.7;
    color: #cbd5e1;
    margin-bottom: 18px;
  }

  .desc em {
    color: #f0abfc;
    font-style: normal;
  }

  /* 功能网格 */
  .grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 10px;
    margin-bottom: 18px;
  }

  .grid-item {
    border: 1px solid rgba(125, 211, 252, 0.25);
    border-radius: 14px;
    background: rgba(255, 255, 255, 0.03);
    padding: 14px 6px;
    text-align: center;
    color: #e0f2fe;
    text-decoration: none;
    transition: all 0.25s ease;
  }

  .grid-item:hover {
    border-color: #a78bfa;
    background: rgba(167, 139, 250, 0.12);
    transform: translateY(-2px);
    box-shadow: 0 6px 18px rgba(167, 139, 250, 0.25);
  }

  .grid-item .icon {
    font-size: 22px;
    margin-bottom: 6px;
    display: block;
  }

  .grid-item .label {
    font-size: 12px;
    color: #c7d2fe;
  }

  /* 文章列表 */
  .section-title {
    font-size: 14px;
    color: #7dd3fc;
    margin-bottom: 10px;
    letter-spacing: 2px;
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .section-title::before {
    content: "";
    width: 4px;
    height: 14px;
    background: linear-gradient(180deg, #7dd3fc, #c084fc);
    border-radius: 2px;
  }

  .post-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
    margin-bottom: 18px;
  }

  .post-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 14px;
    border: 1px solid rgba(148, 163, 184, 0.2);
    border-radius: 12px;
    background: rgba(255, 255, 255, 0.03);
    text-decoration: none;
    color: #e2e8f0;
    transition: all 0.25s ease;
  }

  .post-item:hover {
    border-color: #a78bfa;
    background: rgba(167, 139, 250, 0.1);
  }

  .post-item .post-name {
    font-size: 14px;
  }

  .post-item .post-date {
    font-size: 12px;
    color: #94a3b8;
  }

  /* 主按钮 */
  .btn-primary {
    display: block;
    text-align: center;
    padding: 14px;
    border-radius: 14px;
    background: linear-gradient(135deg, #4f46e5, #7c3aed);
    color: #ffffff;
    font-size: 15px;
    font-weight: bold;
    text-decoration: none;
    letter-spacing: 2px;
    box-shadow: 0 8px 24px rgba(124, 58, 237, 0.45);
    margin-bottom: 12px;
    transition: all 0.25s ease;
  }

  .btn-primary:hover {
    filter: brightness(1.15);
    transform: translateY(-2px);
  }

  .btn-secondary {
    display: block;
    text-align: center;
    padding: 12px;
    border-radius: 14px;
    border: 1px solid rgba(125, 211, 252, 0.4);
    background: rgba(125, 211, 252, 0.08);
    color: #7dd3fc;
    font-size: 14px;
    text-decoration: none;
    letter-spacing: 2px;
    transition: all 0.25s ease;
  }

  .btn-secondary:hover {
    background: rgba(125, 211, 252, 0.18);
    box-shadow: 0 6px 18px rgba(125, 211, 252, 0.25);
  }

  /* 底部状态 */
  .footer-tip {
    text-align: center;
    font-size: 12px;
    color: #64748b;
    margin-top: 10px;
  }
</style>
</head>
<body>

<div class="container">

  <!-- 顶部状态栏 -->
  <div class="status-bar">
    <div class="status-bar-item">
      <div class="label">STATUS</div>
      <div class="value">CTF TRAINING</div>
    </div>
    <div class="status-bar-item">
      <div class="label">ROOM</div>
      <div class="value">WEB</div>
    </div>
    <div class="status-bar-item">
      <div class="label">LEVEL</div>
      <div class="value">BEGINNER</div>
    </div>
  </div>

  <!-- 主卡片 -->
  <div class="card">
    <h1 class="hero-title">
      <div class="line-1">Ready for the</div>
      <div class="line-2">Cybersecurity World</div>
    </h1>
    <div class="hero-sub">> I'm here. 紫薯公主 online.</div>

    <div class="code-box">
      <div><span class="cmd">$</span> whoami</div>
      <div>zishu_princess</div>
      <div><span class="cmd">$</span> cat mission.log</div>
      <div>Web安全入门 · 漏洞复现 · Writeup记录</div>
    </div>

    <p class="desc">
      「以代码为盾，以逻辑为矛」<br>
      从底层理解攻击，用笔记构建防御。<br>
      <em>这里是我的 CTF 作战室。</em>
    </p >

    <!-- 功能入口 -->
    <div class="grid">
      <a href=" " class="grid-item">
        <span class="icon">📘</span>
        <span class="label">Writeup</span>
      </a >
      <a href="#web" class="grid-item">
        <span class="icon">🌐</span>
        <span class="label">Web</span>
      </a >
      <a href="#tools" class="grid-item">
        <span class="icon">🛠️</span>
        <span class="label">Tools</span>
      </a >
      <a href="#about" class="grid-item">
        <span class="icon">👑</span>
        <span class="label">关于</span>
      </a >
    </div>
  </div>

  <!-- 文章列表 -->
  <div class="section-title">LATEST WRITEUP</div>

  <div class="post-list">
    <a href="./2026/09/21/ctfhub-no-verify.html" class="post-item">
      <span class="post-name">Ctfhub 无验证</span>
      <span class="post-date">Sep 21</span>
    </a >
    <a href="./2026/09/19/ctfhub-weak-password.html" class="post-item">
      <span class="post-name">Ctfhub 弱口令</span>
      <span class="post-date">Sep 19</span>
    </a >
    <a href="./2026/09/19/ctfhub-vim-cache.html" class="post-item">
      <span class="post-name">Ctfhub Vim缓存</span>
      <span class="post-date">Sep 19</span>
    </a >
    <a href="./2026/09/19/ctfhub-bak.html" class="post-item">
      <span class="post-name">Ctfhub Bak文件</span>
      <span class="post-date">Sep 19</span>
    </a >
  </div>

  <!-- 按钮 -->
  <a href="#posts" class="btn-primary">进入 CTF 作战室</a >
  <a href="#about" class="btn-secondary">阅读我的第一篇 Writeup</a >

  <div class="footer-tip">
   紫薯公主's CTF Room · Keep hacking.
  </div>

</div>

</body>
</html>
