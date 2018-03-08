---          
layout:     post
title:      添加 mermaid
subtitle:   给 github.io 添加流程图支持
date:       2018-03-6 10:00:00
author:     mzl 
catalog:    true
tags:        
    - mermaid
    - diagram 
---          
             
{:toc}       
# 流程图

最近分析 spark 源码，感觉需要画流程图才能理清流程。但是，之前我的 github.io 不支持画流程，干脆趁机会添加一个好了。
由于 markdown 流程图规范不一致，所以 github.io 不支持流程图 plugin（貌似也不是不可能，我没试过），干脆靠 js 好了。
于是发现了 [mermaid](!https://github.com/knsv/mermaid)。
## 使用方法
使用方法很简单：
1. 在本项目的 styles/js 目录下创建文件 mermaid.min.js，内容是[mermaid.min.js](!https://unpkg.com/mermaid@7.1.2/dist/mermaid.min.js)
2. 在本项目的 _include/footer.html 文件中，添加内容 `<script src="{{ "/styles/js/mermaid.min.js " | prepend: site.baseurl }}"></script>`
3. 测试下是否生效。
4. 更多用法，见 mermaid 的 [gitbook](!https://mermaidjs.github.io/)

<div class="mermaid">
graph TD; 
    A-->B;
    A-->C;
    B-->D;
    C-->D;                                                                                               
</div> 
