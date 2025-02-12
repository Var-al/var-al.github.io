---
layout: post
title: 测试博客
date:   2020-2-12 14:07
description: 让我们尝试一下
toc: true
comments: true
tags:
 - test
---

# FontAssetCreator 字段说明

## 字段名称及作用

| 字段名称              | 作用                                                         |
| --------------------- | ------------------------------------------------------------ |
| **SourceFontFile**    | 选择用于生成 TextMesh Pro 字体资产的字体文件。(在此之内选字符生成) |
| **SamplingPointSize** | 设置用于生成字体纹理的字体大小，以点数为单位。               |
| **Padding**           | 指定字体纹理中字符之间的像素间距。                           |
| **PackingMethod**     | 指定如何将字符放入字体纹理。                                 |
| **AtlasResolution**   | 设置字体纹理的大小宽度和高度，以像素为单位。                 |
| **CharacterSet**      | 选择预定义的字符集。                                         |
| **SelectFontAsset**   | 选择现有的字体资产，以便包含其所有字符。                     |
| **CharacterSequence** | 输入一系列十进制值或值范围，以指定要包括的字符。             |
| **RenderMode**        | 模式来渲染字体图集。                                         |
| **GetKerningPairs**   | 从字体复制字距数据。                                         |

## 字符集选项及附加字段

### CharacterSet 选项

- **ASCII**：包括 ASCII 字符集中的可见字符。
- **Extended ASCII**：包括扩展 ASCII 字符集中的可见字符。
- **ASCII Lowercase**：仅包含 ASCII 字符集中可见的小写字符。
- **ASCII Uppercase**：仅包含 ASCII 字符集中可见的大写字符。
- **Numbers + Symbols**：仅包括来自 ASCII 字符集的可见数字和符号。
- **Custom Range**：输入一系列十进制值或值范围，以指定要包括的字符。
- **Unicode Range (Hex)**：输入一系列 unicode 十六进制值或值范围，以指定要包含的字符。
- **Custom Characters**：输入字符序列以指定要包含的字符。
- **Characters from File**：选择一个包含字符的文件，以指定要包含的字符。

### 附加字段

- **Character File**：当选择“Characters from File”时，用于选择包含字符的文件。
- **Character Sequence**：当选择“Custom Range”或“Unicode Range (Hex)”时，可以输入一系列值或值范围来指定要包含的字符。

## 渲染模式选项

### RenderMode 选项

- **SMOOTH**：将图集渲染为抗锯齿位图，提供更平滑的边缘。
- **RASTER**：将图集渲染为无抗锯齿位图，边缘可能不够平滑。
- **SMOOTH_HINTED**：将图集渲染为抗锯齿位图，同时将字符像素与纹理像素对齐。
- **RASTER_HINTED**：将图集渲染为无抗锯齿位图，并对齐字符像素和纹理像素。
- **SDF**：使用较慢但更准确的SDF生成模式，不进行超采样。
- **SDF8**：使用较慢但更准确的SDF生成模式，8x超采样。
- **SDF16**：使用较慢但更准确的SDF生成模式，16x超采样。
- **SDF32**：使用较慢但更准确的SDF生成模式，32x超采样，适用于复杂或较小的字符。
- **SDFAA**：使用较快但相对不太精确的SDF生成模式，适用于大多数情况。
- **SDFAA_HINTED**：使用较快但相对不太精确的SDF生成模式，并对齐字符像素，适用于大多数情况。

这些字段和选项允许用户根据需要自定义字体资产，以包含所需的字符和样式，并控制字体的渲染效果。