---
layout: post
title: "Abagen 新手教程：安装、兼容性与常见问题全解"
date: 2025-10-22
categories: [Neuroimaging]
tags: [Abagen, Pandas, BrainAtlas, FreeSurfer]
lang: zh
author: "Yangyi Luo"
---

详细讲解如何在新版 Python 与 Pandas 环境中使用 Abagen，解决 set_axis、append、KeyError '[0] not found in axis' 等常见问题。<!--more-->

##  简介

[`abagen`](https://github.com/netneurolab/abagen) 是一个用于从 **Allen Human Brain Atlas (AHBA)** 下载与处理人脑基因表达数据的 Python 包。

不过该包的最新版本仍停留在 **v0.1.3（2021年6月18日）**，这意味着它可能与较新的 Python 与 Pandas 版本存在兼容性问题。

本文将介绍 `abagen` 的使用过程中常见错误的原因与解决方案。

 **官方用户手册**：  
[https://abagen.readthedocs.io/en/stable/usage.html](https://abagen.readthedocs.io/en/stable/usage.html)



**环境信息**

| 项目 | 版本 |
|------|------|
| Python | 3.12.9 |
| pandas | 2.2.3 |
| numpy | 2.2.4 |

主要依赖库：

```bash
pip install abagen pandas nibabel numpy nilearn
```

---

## 一、安装 Abagen

```bash
pip install abagen
```

安装完成后可直接导入：

```python
import abagen
```

如果导入后无报错，即表示安装成功。



## 二、常见问题与修复方法

在使用 `abagen` 处理基因表达数据时，常见错误多与 **pandas 新旧版本差异** 有关。以下是两个典型案例及修复方案。


###  1. `DataFrame.set_axis()` 报错

**错误代码：**

```python
micro = micro.set_axis(symbols, axis=1, inplace=False)
```

**报错信息：**

```
TypeError: set_axis() got an unexpected keyword argument 'inplace'
```

**原因：**  `pandas` 新版本中，`set_axis()` 不再支持 `inplace` 参数，默认返回新的 DataFrame。



**解决方法：** 删除 `inplace` 参数即可兼容新版本。

```python
micro = micro.set_axis(symbols, axis=1)
```




###  2. `DataFrame.append()` 报错

**报错信息：**

```
AttributeError: 'DataFrame' object has no attribute 'append'
```

**原因：** 在 `pandas ≥2.0` 中，`append()` 方法已被移除。


**解决方案 1：添加兼容性补丁**（更推荐这个方法！）

```python
import pandas as pd

# 为新版 pandas 添加旧版 append 方法兼容支持
if not hasattr(pd.DataFrame, "append"):
    pd.DataFrame.append = lambda self, other, **kwargs: pd.concat([self, other], **kwargs)
```

**解决方案 2：降级 pandas**

```bash
pip install pandas==1.4.4
```




### 3.`KeyError: '[0] not found in axis'` 报错

当执行以下代码时，尤其是atlas是一个surface atlas 时，可能会发生以下错误：

```python
abagen.images.check_atlas(
    atlas['image'], atlas['info'], 
    geometry='surface', space='fsaverage5'
)

abagen.get_expression_data(
    atlas=atlas['image'],
    atlas_info=atlas['info'],
    donors = 'all'
)
```

可能会出现：

```
KeyError: '[0] not found in axis'
```



**原因：** 该行假设label.gii中存在 ID = 0（通常代表 “unknown” 或 “medial wall” 区域）。  
但某些模板（例如 fsaverage5 或自定义表面分区）中，该 ID 已被移除，导致无法删除 `[0]`。

该错误出现在函数 `abagen.utils.labeltable_to_df()` 的最后一行：

```python
info = info.set_index('id').drop([0], axis=0).sort_index()
```

**解决方法：** 修改源码，添加安全判断

在abagen安装路径下找到并打开 utils.py

```
.../site-packages/abagen/utils.py
```

找到函数：

```python
def labeltable_to_df(labels):
```

将以下内容：

```python
info = info.set_index('id').drop([0], axis=0).sort_index()
```

替换为安全版本：

```python
info = info.set_index('id')
if 0 in info.index:
    info = info.drop([0], axis=0)
info = info.sort_index()
```

保存后重新运行，即可解决该问题。
