---
layout: post
title: "Abagen - 使用APARC.A2009S (Destrieux) Atlas 创建 Parcellation"
date: 2025-10-23
categories: [Neuroimaging]
tags: [Abagen, Gene Expression, Python]
lang: zh
author: "Yangyi Luo"
---
详细讲解如何使用 Abagen 处理 Allen Brain Atlas 基因表达数据创建 Parcellation并生成基因表达矩阵，尤其是APARC.A2009S (Destrieux) Atlas 。<!--more-->

# Desikan-Killiany atlas 上的基因表达矩阵
## 1.使用体积分区(load the volumetric atlas )


```python
import abagen
import nibabel as nib
from abagen import images, datasets
import os
import numpy as np
import pandas as pd

# 添加对旧版pandas append方法的兼容性支持
if not hasattr(pd.DataFrame, "append"):
    pd.DataFrame.append = lambda self, other, **kwargs: pd.concat([self, other], **kwargs)

BASE_DIR = os.getcwd()
data_path = os.path.join(BASE_DIR, "data")
```


```python
atlas = abagen.fetch_desikan_killiany()
print(atlas['image'])
print(atlas['info'])
```

    c:\Users\Lenovo\.conda\envs\mrienv\Lib\site-packages\abagen\data\atlas-desikankilliany.nii.gz
    c:\Users\Lenovo\.conda\envs\mrienv\Lib\site-packages\abagen\data\atlas-desikankilliany.csv



```python
atlas = images.check_atlas(atlas['image'], atlas['info'])
print(atlas)
```

    AtlasTree[n_rois=83, n_voxel=819621]


下载完微阵列数据并选择分区后，就可以使用 abagen.get_expression_data 获取基因表达矩阵数据了。


```python
expression = abagen.get_expression_data(atlas['image'], atlas['info'])
print(expression.shape)
```

    (83, 15633)

## 2.使用表面分区(load the surface version of the atlas)


```python
atlas = abagen.fetch_desikan_killiany(surface=True)
print(atlas['image'])
print(atlas['info'])
```

    ('c:\\Users\\Lenovo\\.conda\\envs\\mrienv\\Lib\\site-packages\\abagen\\data\\atlas-desikankilliany-lh.label.gii.gz', 'c:\\Users\\Lenovo\\.conda\\envs\\mrienv\\Lib\\site-packages\\abagen\\data\\atlas-desikankilliany-rh.label.gii.gz')
    c:\Users\Lenovo\.conda\envs\mrienv\Lib\site-packages\abagen\data\atlas-desikankilliany.csv


表面分区 atlas['image'] 是一个元组，具有两个元素 ('atlas-desikankilliany-lh.label.gii.gz','atlas-desikankilliany-rh.label.gii.gz')
atlas['info']和体积分区时使用的info表没有区别。


```python
atlas_check = images.check_atlas(atlas['image'], atlas['info'])
print(atlas_check)
```

    AtlasTree[n_rois=68, n_vertex=18426]



```python
atlas_file = list(atlas['image'])
print("Atlas files:", atlas_file)
for f in atlas_file:
    atlas_data = nib.load(f).darrays[0].data
    unique_labels = np.unique(atlas_data)
    print(f"{f} unique labels:", unique_labels)
```

    Atlas files: ['c:\\Users\\Lenovo\\.conda\\envs\\mrienv\\Lib\\site-packages\\abagen\\data\\atlas-desikankilliany-lh.label.gii.gz', 'c:\\Users\\Lenovo\\.conda\\envs\\mrienv\\Lib\\site-packages\\abagen\\data\\atlas-desikankilliany-rh.label.gii.gz']
    c:\Users\Lenovo\.conda\envs\mrienv\Lib\site-packages\abagen\data\atlas-desikankilliany-lh.label.gii.gz unique labels: [ 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23
     24 25 26 27 28 29 30 31 32 33 34]
    c:\Users\Lenovo\.conda\envs\mrienv\Lib\site-packages\abagen\data\atlas-desikankilliany-rh.label.gii.gz unique labels: [ 0 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64
     65 66 67 68 69 70 71 72 73 74 75]



```python
atlas_info = pd.read_csv(atlas['info'])
print(atlas_info)
```

        id                    label hemisphere            structure
    0    1                 bankssts          L               cortex
    1    2  caudalanteriorcingulate          L               cortex
    2    3      caudalmiddlefrontal          L               cortex
    3    4                   cuneus          L               cortex
    4    5               entorhinal          L               cortex
    ..  ..                      ...        ...                  ...
    78  79                 pallidum          R  subcortex/brainstem
    79  80            accumbensarea          R  subcortex/brainstem
    80  81              hippocampus          R  subcortex/brainstem
    81  82                 amygdala          R  subcortex/brainstem
    82  83                brainstem          B  subcortex/brainstem
    
    [83 rows x 4 columns]


在这里我们可以观察到，label.gii 文件的编号和 csv文件是可以匹配的，尽管csv中有83个label，但是lh.label.gii有34个，rh.label.gii 有33个，加起来不足83个。但是label.gii中的编号在CSV文件中都能找到对应区域即可。


```python

atlas = datasets.fetch_desikan_killiany(surface=True)
expression1 = abagen.get_expression_data(
    atlas=atlas['image'],
    atlas_info=atlas['info'],
    donors = 'all'
)
print(expression1.shape)
```

    (68, 15633)




# 使用自己的 Atlas 创建脑区分区（Parcellation）
如果你想使用其他的分区（atlas），需要准备两个文件：

1. **Atlas 文件**（`.nii` 或 `.gii` 格式）  
   - 如果是 **体积型（volumetric）atlas**，请使用 **NIfTI (`.nii`) 格式**  
   - 如果是 **表面型（surface）atlas**，请使用 **GIFTI (`.gii`) 格式**  
     > 官方提醒：表面型 atlas 的分辨率建议为 **fsaverage5**  

2. **信息文件（CSV 格式）**  
   - 对于 `csv` 文件，可以从 `.annot` 文件中提取。如果你想提供自定义的 CSV 文件，需要确保文件至少包含脑区 ID、半球信息、结构类别等  

- `id`：对应 atlas 图像中标签的整数 ID  
- `hemisphere`：半球信息（左/右/双侧），用 `'L'`、`'R'` 或 `'B'` 表示  
- `structure`：大脑结构类别，例如：`'cortex'`,`'subcortex/brainstem'`,`'cerebellum'`,`'white matter'`,`'other'`


## 使用 APARC.A2009S (Destrieux) Atlas 创建 Parcellation

以 **APARC.A2009S（Destrieux）atlas** 为例，这是一个 **表面（surface）大脑模板**，所以我们需要获取以下文件：

- **左半球标签文件**：`lh.aparc.a2009s.label.gii`  
- **右半球标签文件**：`rh.aparc.a2009s.label.gii`  
- **对应的 info.csv 文件**

> **注意**：FreeSurfer 通常不会直接提供 `.label.gii` 文件，但可以通过转换获得。`abagen` 提供了函数 `abagen.annot_to_gifti()`，可以将 FreeSurfer 的 `.annot` 文件转换为 `.gii` 文件。  
> 文档链接：[abagen.annot_to_gifti](https://abagen.readthedocs.io/en/stable/generated/abagen.annot_to_gifti.html#abagen.annot_to_gifti)

> 我们使用的这些文件都是基于 **FreeSurfer 的 fsaverage5 模板** 下的文件。


要获取 **基因表达矩阵**，需要用到两个重要的 `abagen` 函数：

1. **`abagen.images.check_atlas()`**  
   - 用于检查你的 atlas 文件格式是否正确  
   - 官方文档：[check_atlas](https://abagen.readthedocs.io/en/stable/generated/abagen.check_atlas.html#abagen.check_atlas)

2. **`abagen.get_expression_data()`**  
   - 将 AHBA 原始基因探针数据映射到脑区，输出 **脑区 × 基因** 的表达矩阵  
   - 官方文档：[get_expression_data](https://abagen.readthedocs.io/en/stable/generated/abagen.get_expression_data.html#abagen.get_expression_data)



```python
# 从annot中提取csv文件
 
out_csv = os.path.join(data_path, "aparc_a2009s_info.csv")

def extract_info(annot_path, hemisphere):
    labels, ctab, names = nib.freesurfer.read_annot(annot_path)
    df = pd.DataFrame({
        "id": list(range(len(names))),
        "label": [n.decode("utf-8") for n in names],
        "hemisphere": hemisphere,
        "structure": ["cortex"] * len(names)
    })
    # 移除 FreeSurfer 自动添加的 'unknown'
    df = df[df["label"] != "unknown"]
    return df

# 左右半球
df_l = extract_info(os.path.join(data_path, "lh.aparc.a2009s.annot"), "L")
df_r = extract_info(os.path.join(data_path, "rh.aparc.a2009s.annot"), "R")

atlas_info = pd.concat([df_l, df_r], ignore_index=True)
atlas_info.to_csv(out_csv, index=False)

print(f"✅ Saved atlas info to: {out_csv}")
print(atlas_info)
```

    ✅ Saved atlas info to: c:\Users\Lenovo\Desktop\freesurfer\data\aparc_a2009s_info.csv
         id                  label hemisphere structure
    0     0                Unknown          L    cortex
    1     1   G_and_S_frontomargin          L    cortex
    2     2  G_and_S_occipital_inf          L    cortex
    3     3    G_and_S_paracentral          L    cortex
    4     4     G_and_S_subcentral          L    cortex
    ..   ..                    ...        ...       ...
    147  71           S_suborbital          R    cortex
    148  72          S_subparietal          R    cortex
    149  73         S_temporal_inf          R    cortex
    150  74         S_temporal_sup          R    cortex
    151  75  S_temporal_transverse          R    cortex
    
    [152 rows x 4 columns]


由于是从左右两个半球中分别提取的，所以info.csv中，编号为0-75，然后又是0-75。后续对编号进行转换成为1-150。



从./freesurfer/subjects/fsaverage5目录取出以下文件：

标签文件：
- label/lh.aparc.a2009s.annot
- label/rh.aparc.a2009s.annot

对应的 fsaverage surface：
- surf/lh.pial
- surf/rh.pial

## 检查 Atlas 格式
使用abagen.images.check_atlas() 辅助检查确保格式匹配，这里由于是surface atlas,需要用到lh.surf.gii，rh.surf.gii。这两个文件可以使用freesurfer自带的命令将 `.pial` 文件转换得到，例如：
```bash
mris_convert ./freesurfer/subjects/fsaverage5/surf/lh.pial \
             ./freesurfer/subjects/fsaverage5/surf/lh.surf.gii
```
我也已将转换好的 GIFTI 文件放在文末，方便直接下载使用。


```python
# annot转换成label.gii
lh_annot = os.path.join(data_path, "lh.aparc.a2009s.annot")
rh_annot = os.path.join(data_path, "rh.aparc.a2009s.annot")

lh_gifti = abagen.annot_to_gifti(lh_annot)
rh_gifti = abagen.annot_to_gifti(rh_annot)

#保存
lh_gifti_path = os.path.join(data_path, "lh.aparc.a2009s.label1.gii")
rh_gifti_path = os.path.join(data_path, "rh.aparc.a2009s.label1.gii")
nib.save(lh_gifti, lh_gifti_path)
nib.save(rh_gifti, rh_gifti_path)

print("Left hemisphere GIFTI:", lh_gifti)
print("Right hemisphere GIFTI:", rh_gifti)

gifti_paths = (lh_gifti, rh_gifti)
```

    Left hemisphere GIFTI: <nibabel.gifti.gifti.GiftiImage object at 0x000001FFA4B2F020>
    Right hemisphere GIFTI: <nibabel.gifti.gifti.GiftiImage object at 0x000001FFA4A99FA0>



```python
#检查顶点数是否匹配，确保都是fsaverage5的分辨率
surf_path = os.path.join(data_path, "lh.surf.gii")
surf = nib.load(surf_path)
label_path = os.path.join(data_path, "lh.aparc.a2009s.label.gii")
label = nib.load(label_path)

print(len(surf.darrays[0].data))  # 顶点数量
print(len(label.darrays[0].data)) # label 数量（应完全一致）

```

    10242
    10242


现在我们获得了fsaverage5下的surf.gii和label.gii文件，以及info.csv文件，接下来就是将info.csv文件和label.gii编号进行匹配。


```python
import os
import numpy as np
import pandas as pd
import nibabel as nib

# --- 文件路径 ---
csv_path = os.path.join(data_path, "aparc_a2009s_info.csv")
lh_label_path = os.path.join(data_path, "lh.aparc.a2009s.label.gii")
rh_label_path = os.path.join(data_path, "rh.aparc.a2009s.label.gii")

# --- 读取 CSV ---
info = pd.read_csv(csv_path)
print("Columns:", info.columns)

# 删除未知和无效编号
info = info[~info["id"].isin([-1])]
info = info[~((info["hemisphere"] == "R") & (info["label"] == "Unknown"))]


# 分半球
info_l = info[info["hemisphere"].str.lower().str.startswith("l")].copy()
info_r = info[info["hemisphere"].str.lower().str.startswith("r")].copy()

# --- 重新编号 ---
info_l["new_label"] = np.arange(0, len(info_l))
info_r["new_label"] = np.arange(76, 76 + len(info_r))

# 合并半球信息
info_new = pd.concat([info_l, info_r], ignore_index=True)



# --- 修改 label.gii 文件 ---
def remap_label_gii(label_path, info_old):
    print("infoold:",info_old)
    print(f"正在处理: {label_path}")
    img = nib.load(label_path)
    data = img.darrays[0].data.copy()
    
    unique_labels = np.unique(data)
    print("原始唯一 label 值:", unique_labels)
    
    # 使用 id -> new_label 作为映射
    label_map = dict(zip(info_old["id"], info_old["new_label"]))  
    
    # 替换
    new_data = np.vectorize(lambda x: label_map.get(x, 0))(data)
    
    print("修改后唯一 label 值:", np.unique(new_data))
    
    new_img = nib.gifti.GiftiImage(darrays=[nib.gifti.GiftiDataArray(new_data.astype(np.int32))])
    new_path = label_path.replace(".label.gii", "_ordered.label.gii")
    nib.save(new_img, new_path)
    print(f"已保存: {new_path}")
    return new_path

# 分别处理左右半球
lh_new = remap_label_gii(lh_label_path, info_l)
rh_new = remap_label_gii(rh_label_path, info_r)


# 先删掉旧id列、重命名 new_label → id
info_new = info_new.drop(columns=["id"])
info_new = info_new.rename(columns={"new_label": "id"})


# 保存
new_csv_path = os.path.splitext(csv_path)[0] + "_updated.csv"
info_new.to_csv(new_csv_path, index=False)
print(f"新的 parcellation 信息已保存至: {new_csv_path}")
```

    Columns: Index(['id', 'label', 'hemisphere', 'structure'], dtype='object')
    infoold:     id                  label hemisphere structure  new_label
    0    0                Unknown          L    cortex          0
    1    1   G_and_S_frontomargin          L    cortex          1
    2    2  G_and_S_occipital_inf          L    cortex          2
    3    3    G_and_S_paracentral          L    cortex          3
    4    4     G_and_S_subcentral          L    cortex          4
    ..  ..                    ...        ...       ...        ...
    71  71           S_suborbital          L    cortex         71
    72  72          S_subparietal          L    cortex         72
    73  73         S_temporal_inf          L    cortex         73
    74  74         S_temporal_sup          L    cortex         74
    75  75  S_temporal_transverse          L    cortex         75
    
    [76 rows x 5 columns]
    
    正在处理:.\data\lh.aparc.a2009s.label.gii
    原始唯一 label 值: [ 1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24
     25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48
     49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72
     73 74 75]
    修改后唯一 label 值: [ 1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24
     25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48
     49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72
     73 74 75]
    已保存: .\data\lh.aparc.a2009s_ordered.label.gii
    infoold:      id                     label hemisphere structure  new_label
    77    1      G_and_S_frontomargin          R    cortex         76
    78    2     G_and_S_occipital_inf          R    cortex         77
    79    3       G_and_S_paracentral          R    cortex         78
    80    4        G_and_S_subcentral          R    cortex         79
    81    5  G_and_S_transv_frontopol          R    cortex         80
    ..   ..                       ...        ...       ...        ...
    147  71              S_suborbital          R    cortex        146
    148  72             S_subparietal          R    cortex        147
    149  73            S_temporal_inf          R    cortex        148
    150  74            S_temporal_sup          R    cortex        149
    151  75     S_temporal_transverse          R    cortex        150
    
    [75 rows x 5 columns]
    
    正在处理: .\data\rh.aparc.a2009s.label.gii
    原始唯一 label 值: [ 1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24
     25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48
     49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72
     73 74 75]
    修改后唯一 label 值: [ 76  77  78  79  80  81  82  83  84  85  86  87  88  89  90  91  92  93
      94  95  96  97  98  99 100 101 102 103 104 105 106 107 108 109 110 111
     112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127 128 129
     130 131 132 133 134 135 136 137 138 139 140 141 142 143 144 145 146 147
     148 149 150]
    已保存: .\rh.aparc.a2009s_ordered.label.gii
    新的 parcellation 信息已保存至: .\aparc_a2009s_info_updated.csv


使用images.check_atlas来检查格式是否符合要求


```python
datlas = {}
lh_surf  = os.path.join(data_path, "lh.surf.gii")
rh_surf  = os.path.join(data_path, "rh.surf.gii")
geometry = (lh_surf, rh_surf)

lh_new_label_path = os.path.join(data_path, "lh.aparc.a2009s_ordered.label.gii")
rh_new_label_path = os.path.join(data_path, "rh.aparc.a2009s_ordered.label.gii")

datlas['image']= (lh_new_label_path,rh_new_label_path)
datlas['info']= os.path.join(data_path, "aparc_a2009s_info_updated.csv")

datlas = images.check_atlas(datlas['image'], datlas['info'],geometry=geometry,space='fsaverage5')
print(datlas)
```

    AtlasTree[n_rois=150, n_vertex=20484]





```python
atlas = {}
lh_new_label_path = os.path.join(data_path, "lh.aparc.a2009s_ordered.label.gii")
rh_new_label_path = os.path.join(data_path, "rh.aparc.a2009s_ordered.label.gii")

atlas['image']= (lh_new_label_path,rh_new_label_path)
atlas['info']= os.path.join(data_path, "aparc_a2009s_info_updated.csv")

expression = abagen.get_expression_data(
    atlas=atlas['image'],
    atlas_info=atlas['info'],
    donors = 'all'
)
```

```python
print(expert.shape)
print(expert)
save_path = os.path.join(data_path, "gene_expression_data.csv")
expert.to_csv(save_path)
```

    (150, 15632)
    gene_symbol      A1BG  A1BG-AS1       A2M     A2ML1   A3GALT2    A4GALT  \
    label                                                                     
    1            0.722862  0.400509  0.364763  0.513018  0.495997  0.441195   
    2            0.408178  0.433791  0.595114  0.408449  0.426421  0.558950   
    3            0.317764  0.367745  0.487841  0.676806  0.577220  0.418968   
    4            0.416215  0.531087  0.485960  0.598213  0.482999  0.552229   
    5            0.218712  0.254797  0.275308  0.559191  0.277103  0.465715   
    ...               ...       ...       ...       ...       ...       ...   
    146          0.819563  0.575742  0.256018  0.255194  0.768340  0.862924   
    147          0.486586  0.290828  0.383218  0.597011  0.386646  0.819549   
    148          0.694668  0.546576  0.463103  0.575538  0.534532  0.470500   
    149          0.648712  0.599177  0.542625  0.482751  0.476349  0.480231   
    150               NaN       NaN       NaN       NaN       NaN       NaN   

    gene_symbol      AAAS      AACS   AADACL3     AADAT  ...      ZW10    ZWILCH  \
    label                                                ...                       
    1            0.411969  0.288477  0.630404  0.332772  ...  0.152940  0.531106   
    2            0.596045  0.577286  0.403251  0.485811  ...  0.496530  0.615381   
    3            0.622825  0.573047  0.598773  0.553653  ...  0.552615  0.352953   
    4            0.507437  0.510832  0.454590  0.589677  ...  0.496594  0.542945   
    5            0.478031  0.295714  0.252843  0.951968  ...  0.309147  0.269570   
    ...               ...       ...       ...       ...  ...       ...       ...   
    146          0.201567  0.511338  0.813461  0.170971  ...  0.369177  0.902881   
    147          0.445857  0.390764  0.259320  0.215635  ...  0.423673  0.473919   
    ...
    149          0.512266  0.533650  
    150               NaN       NaN  

    [150 rows x 15632 columns]    
