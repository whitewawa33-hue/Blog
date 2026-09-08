---
title: 网站的markdown写文章指南
published: 2026-08-30
description: "使用markdown写文章的常用功能和示例指南"
tags: [Markdown, Guide]
category: 教程
---
### 文章路径：`D:\TOOL\AI\claude-code\projects\blog\template\Blog\src\content`
# 第一部分 Frontmatter（元数据）
### >示例
```
---
title: 网站的markdown写文章指南
published: 2026-08-30
description: "使用markdown写文章的常用功能和示例指南"
tags: [Markdown, guide]
category: 教程
password: "114514"
---
```

# 第二部分：正文内容
## 常用语法
| 语法 | 效果 | 
| :--- | :--- | 
| `# 标题` | 一级标题 | 
| `## 标题` | 二级标题 | 
| `**加粗**` | 加粗文字 | 
| `*斜体*` | 斜体文字 |
| `- 项目` | 无序列表 | 
| `1. 项目` | 有序列表 | 
| `>正文` | 引用块 |
| `` `代码` `` | 行内代码 |
| `==正文==` | 高亮文字 |

# 第三部分：引用
## 1.图片
### >示例  
```
![文字](图片路径)
```
## 2.文章
### >示例  

```
[[MYblog|（自定义内容）]]
```
### >效果  
[[MYblog|自定义内容]]

# 第四部分：Mermaid 图表
### 示例
````
```mermaid
graph TD
    A[开始] --> B{条件检查}
    B -->|是| C[处理步骤 1]
    B -->|否| D[处理步骤 2]
    C --> E[子过程]
    D --> E
    subgraph E [子过程详情]
        E1[子步骤 1] --> E2[子步骤 2]
        E2 --> E3[子步骤 3]
    end
    E --> F{另一个决策}
    F -->|选项 1| G[结果 1]
    F -->|选项 2| H[结果 2]
    F -->|选项 3| I[结果 3]
    G --> J[结束]
    H --> J
    I --> J
```
````

### 效果
```mermaid
graph TD
    A[开始] --> B{条件检查}
    B -->|是| C[处理步骤 1]
    B -->|否| D[处理步骤 2]
    C --> E[子过程]
    D --> E
    subgraph E [子过程详情]
        E1[子步骤 1] --> E2[子步骤 2]
        E2 --> E3[子步骤 3]
    end
    E --> F{另一个决策}
    F -->|选项 1| G[结果 1]
    F -->|选项 2| H[结果 2]
    F -->|选项 3| I[结果 3]
    G --> J[结束]
    H --> J
    I --> J
```

