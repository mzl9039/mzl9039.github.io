---
layout:     post
title:      添加 MathJax.js
subtitle:   
date:       2018-06-05 00:00:00
author:     mzl
catalog:    true
tags:
    - md
    - math
    - MathJax
---

# 通过 js 渲染数学公式

最近在读论文，很多情况下，需要在 markdown 下渲染数学公式，因为在这里记录下在博客中实现数学公式渲染的方法。
百度一把知道目录这种方式，基本上都是用 MathJax.js 完成的，基本方法就是在 head 标签中引入js文件：

```javascript
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
```

如果需要做其它设置，如通过 `\( ... \)` 识别公式等，可以添加如下配置：

```javascript
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
    tex2jax: {
        inlineMath: [ ['$','$'], ["\\(","\\)"] ],
        displayMath: [ ['$$','$$'], ["\\[","\\]"] ]
    }
});
</script>
```
## 公式标签

行内公式使用 "$" ... "$" 或 "\\(" ... "\\)" 来表示

独立公式另起一行，并添加一个空行，然后使用 "$$" ... "$$" 或 "\\[" ... "\\]" 来表示

## 测试公式如下：

行内公式1: $\frac {1} {2}$

行内公式2: \\( ... \\)

独立公式:

$$
    J_\alpha(x) = \sum_{m=0}^\infty \frac{(-1)^m}{m! \Gamma (m + \alpha + 1)} {\left({ \frac{x}{2} }\right)}^{2m + \alpha} \text {，独立公式示例}
$$

独立公式2：

$$ 
    f_\epsilon(p_{i,l}, p_{j,k}) = 
    \begin{cases} 
    0 & dist(p_{i,l}, p_{j,k}) > \epsilon \\
    1 - \frac {dist(p_{i,l}, p_{j,k})} {\epsilon} & otherwise
    \end{cases}
$$ 
