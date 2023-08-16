---
layout: page
title: About
comments: true
permalink: /commands/
---

* content
{:toc}

#### 工具类

* 按文件后缀批量转换换行符
  * Linux / MinGW
    ```
    find . -name '*.cs' -exec dos2unix {} \;
    ```
  * Windows:
    ```
    for /R %G in (*.cs) do dos2unix "%G"
    ```
* 按.gitattributes格式化目录
  ```
  git add --renormalize .
  ```
