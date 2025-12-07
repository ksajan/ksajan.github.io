+++
date = '2025-06-04T18:07:33+05:45'
draft = false
title = 'Code post'
authors=['John']
tags=['red', 'white', 'blue', 'hello']
toc=false
+++

{{< codeblock lang="python" >}}
def greet(name):
    print(f"Hello, {name}!")
{{< /codeblock >}}

{{< codeblock lang="html" >}}
{{- $lang := .Get "lang" | default "text" -}}

<style>
  /* codeblock shortcode */
  .custom-code-block-wrapper {
    position: relative;
    margin: 1.5rem 0;
    border-radius: 8px;
    background-color: #1e1e1e;
    overflow: hidden;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.25);
  }

  .custom-code-toolbar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    background-color: #2d2d2d;
    padding: 0.4rem 0.75rem;
    border-bottom: 1px solid #444;
    font-family: monospace;
    font-size: 0.85rem;
  }

  .code-lang-label {
    color: #888;
  }

  .copy-code-btn {
    background: none;
    border: none;
    color: #bbb;
    cursor: pointer;
    transition: color 0.2s ease;
    font-size: 0.9rem;
  }

  .copy-code-btn:hover {
    color: #fff;
  }

  .custom-code-block-wrapper .highlight {
    margin: 0;
  }

  .custom-code-block-wrapper pre {
    margin: 0;
    padding: 1rem;
    overflow-x: auto;
    background-color: #1e1e1e !important;
  }

  .custom-code-block-wrapper code {
    font-family: Consolas, Monaco, "Courier New", monospace;
    font-size: 0.95rem;
    background-color: transparent;
  }
</style>

<div class="custom-code-block-wrapper" data-lang="{{ $lang }}">
  <div class="custom-code-toolbar">
    <span class="code-lang-label">{{ upper $lang }}</span>
    <button class="copy-code-btn" onclick="copyCode(this)">📋 Copy</button>
  </div>
  {{ highlight (trim .Inner "\n\r") $lang "style=monokai" }}
</div>

<script>
  function copyCode(btn) {
    const wrapper = btn.closest(".custom-code-block-wrapper");
    const code = wrapper.querySelector("code").innerText;
    navigator.clipboard.writeText(code);
    btn.textContent = "✅ Copied";
    setTimeout(() => (btn.textContent = "📋 Copy"), 1500);
  }
</script>
{{< /codeblock >}}
