---
layout: post
title: "Jekyll 的 Markdown 與程式碼高亮設定"
date: 2022-01-28 20:15:00 +0800
categories: [心法]
tags: [Jekyll, Markdown]
---

寫技術文章最重要的就是程式碼要好看。Jekyll 預設用 Kramdown 當 Markdown 解析器，支援 GitHub Flavored Markdown，這點很方便。

程式碼區塊用三個反引號包起來，後面加上語言名稱就能語法高亮：

    ```java
    public class Example {
        public static void main(String[] args) {
            System.out.println("Hello World");
        }
    }
    ```

Jekyll 內建 Rouge 作為語法高亮引擎。在 `_config.yml` 確認設定：

```yaml
markdown: kramdown
highlighter: rouge
kramdown:
  syntax_highlighter: rouge
```

Rouge 支援超多語言，Java、Python、JavaScript、Ruby 都沒問題。高亮的配色方案可以自己選，我用的是 `monokai` 風格。

執行這個指令產生 CSS：

```bash
rougify style monokai > assets/css/syntax.css
```

然後在 `default.html` 引入這個 CSS 檔。

另外 Kramdown 還支援表格、註腳、數學公式等進階語法。表格用 pipe 符號分隔欄位，第二行用 dash 分隔標題和內容。

目前測試下來效果不錯，程式碼看起來很清楚。
