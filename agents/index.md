{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# エージェントと LLM {#agents--llms}

これらのガイドラインは丁寧に手作業で書かれていますが、LLM が読むことも念頭に置いて書かれています。
コードを書いたりレビューしたりするときに、エージェントのセッションに含められる要約版を提供しています。

## 利用できるリソース {#available-resources}

| 名前 | サイズ | 説明 |
|------|------|-------------|
| [all](https://microsoft.github.io/rust-guidelines/agents/all.txt) [📋](#) | ~133 kb, ~34k tokens | Pragmatic Rust Guidelines の全体を1つのファイルにまとめたもの。 |

<script>
// 貼り付けボタンを動作させる
document.addEventListener('DOMContentLoaded', function() {
    document.querySelectorAll('a[href="#"]').forEach(function(copyIcon) {
        copyIcon.onclick = function(e) {
            e.preventDefault();
            const link = this.parentElement.querySelector('a[href*="microsoft.github.com"]');
            if (link) {
                navigator.clipboard.writeText(link.href);
            }
        };
    });
});
</script>
{% endraw %}
