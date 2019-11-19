---
layout:     post
title:      给Jekyll增加Latex公式渲染
subtitle:   
date:       2019-06-18
author:     pandaychen
header-img: 
catalog: true
tags:
    - Latex
---

##  介绍
上一次用[`Latex`](https://zh.wikipedia.org/wiki/LaTeX)都是7年前的事情了，编写密码学的复杂的数学公式之必备利器。

##  Jekyll支持Latex的设置

1.  第一步，将`_config.yml`中的 `markdown` 修改为
``` js
markdown: kramdown
```
2. 第二步，在`header`文件中添加引用和设置代码，也就是_include/header.html中
``` js
  <script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
        inlineMath: [['$','$']]
      }
    });
  </script>
  <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
``` 

##  数学公式的例子
1.	
$$E=mc^2$$
2.	行间公式（Lorentz方程）
$$ 
\begin{aligned} \dot{x} &= \sigma(y-x) \\ 
\dot{y} &= \rho x - y - xz \\ 
\dot{z} &= -\beta z + xy \end{aligned} 
$$
3.	
$$
R_{\mu \nu} - {1 \over 2}g_{\mu \nu}\,R + g_{\mu \nu} \Lambda
= {8 \pi G \over c^4} T_{\mu \nu}
$$


##  参考
-   [How to support latex in github-pages?](https://stackoverflow.com/questions/26275645/how-to-support-latex-in-github-pages)
-   [在Jekyll中使用LaTex](https://lloyar.github.io/2018/10/08/mathjax-in-jekyll.html)
