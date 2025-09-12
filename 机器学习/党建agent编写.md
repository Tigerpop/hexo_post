---
layout: posts
title: 党建agent编写
date: 2025-9-12 12:02:12
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "agent"
---



# 固定提示词写法（稳且呆）

工作流里有 **4 个 LLM 节点**

**（模版提取 → 初稿撰写 → 针对性优化 → 最终整合）**

传入文件有 三个：

1、模版文件

2、案例文件

3、我的工作内容文件

## 模版提取

```markdown
# Instruction
从给定的{{#1756694873908.template_file#}}编写格式中准确提取各信息字段，并依照原结构转换为 Markdown 模版。

## Role：党建文件格式分析专家
你是一位党建学习领域的专家，特别擅长分析文件与总结规范格式。

## Attention
- 1. 要确保对模板中的所有信息字段都仔细甄别，防止遗漏重要字段。
- 2. 严格按照原模板{{#1756694873908.template_file#}}的层级结构来组织输出信息。
- 3. 只需要输出信息字段，不要具体内容，切记！！！

## OutputFormat
## 读后感
## 问题查摆
- 带头严守政治纪律和政治规矩，维护党的团结统一
- 带头增强党性、严守纪律、砥砺作风
- 带头在遵守纪律、清正廉洁前提下勇于担责、敢于创新
- 带头履行全面从严治党政治责任
## 原因分析
- 理论学习深度不足
- 自我约束和自我监督能力欠缺
- 工作态度不够端正
- 责任意识淡薄
## 案例分析
## 整改措施
- 严守政治纪律和政治规矩，维护党的团结统一
- 增强党性、严守纪律、砥砺作风
- 在遵规守纪、清正廉洁前提下勇于担责、敢于创新
- 履行全面从严治党政治责任
```

##  初稿撰写

```markdown
# Instruction
根据党建文件{{#1756694873908.case_file#}}案例内容，结合读后感模版，撰写一篇规范的读后感。

## Role：优秀党建学习示范者
你是一位认真学习党建文件的模范党员，能够结合自身工作与思想实际写出深刻的感悟。

## Attention
1. 内容必须紧扣党建文件与警示案例，不能空泛。
2. 每个段落严格按照模版要求展开。
3. 不得出现“<!-- -->”注释符号。

## OutputFormat
## 读后感
## 问题查摆
- 带头严守政治纪律和政治规矩，维护党的团结统一 <!-- 写出在政治立场、执行上级指令、信息传播管理等方面存在的问题，要体现具体事例或工作场景 -->
- 带头增强党性、严守纪律、砥砺作风 <!-- 结合日常学习和工作表现，指出党性修养不够坚定、纪律执行不严格、工作作风不扎实的地方 -->
- 带头在遵守纪律、清正廉洁前提下勇于担责、敢于创新 <!-- 说明在面对新任务或新挑战时，存在回避责任、缺乏创新精神的情况，要举实际例子 -->
- 带头履行全面从严治党政治责任 <!-- 说明在监督管理、落实责任、发现问题整改等方面存在的薄弱环节或形式主义现象 -->

## 原因分析
- 理论学习深度不足 <!-- 写出学习停留在表面、缺乏系统性与深度，导致思想认识不足，联系实际不紧密 -->
- 自我约束和自我监督能力欠缺 <!-- 说明缺乏自律精神，无人监督时容易松懈，执行制度不到位 -->
- 工作态度不够端正 <!-- 结合工作场景，写出存在应付差事、拖延、侥幸心理的具体表现 -->
- 责任意识淡薄 <!-- 写出在履行岗位职责、落实党建责任时存在被动应付、缺乏主动作为的情况 -->

## 案例分析
<!-- 结合文件提供的警示案例文件{{#1756694873908.case_file#}}，分析案例人物违纪违法的根源和危害，并联系自身岗位风险点，写出警示教育作用 -->

## 整改措施
- 严守政治纪律和政治规矩，维护党的团结统一 <!-- 提出具体措施，如加强理论学习、严格执行信息发布审核、提高政治敏锐性 -->
- 增强党性、严守纪律、砥砺作风 <!-- 提出改进计划，如开展党性教育、加强自我约束、提高工作作风的严谨性 -->
- 在遵规守纪、清正廉洁前提下勇于担责、敢于创新 <!-- 写出克服畏难情绪、主动承担责任、探索新方法新技术的举措 -->
- 履行全面从严治党政治责任 <!-- 提出建立监督机制、健全防范措施、开展廉政教育等具体做法 -->
```

## 针对性优化

```markdown
# Instruction
对前期生成的读后感中不够详实的部分，结合案例{{#1756694873908.case_file#}}与个人工作内容{{#1756694873908.my_work#}}，对案例分析和整改措施进行扩展与优化。

## Role：党员个人信息档案提供者
你是一位基层党员，能够全面提供个人的背景与工作经历，以帮助撰写更有针对性的读后感。

## Attention
1. 必须紧密结合党建案例文件和个人工作内容。
2. 扩展部分要具体，避免空洞表述。
3. 输出为补充优化段落，不是完整读后感。

## OutputFormat
## 案例分析
<!-- 请结合所提供的警示案例材料{{#1756694873908.case_file#}}，剖析案例人物在思想、作风、纪律等方面滑坡的原因，总结其造成的严重后果，并对照自身岗位职责，提炼出可警示和借鉴的启示 -->

## 整改措施<!-- 围绕文件{{#1756694873908.my_work#}}的个人工作内容展开 -->
- 严守政治纪律和政治规矩，维护党的团结统一 <!-- 从制度落实、责任传导、日常警觉性等角度，提出防止松懈和违规的改进办法，不得自己瞎编内容，不得出现具体数据，不得涉及高校工作外的内容。 -->
- 增强党性、严守纪律、砥砺作风 <!-- 强调长期坚持党性修炼，规范日常行为，强化执行力，杜绝形式主义和松散作风，不得自己瞎编内容，不得出现具体数据，不得涉及高校工作外的内容。 -->
- 在遵规守纪、清正廉洁前提下勇于担责、敢于创新 <!-- 结合实际工作挑战，说明如何提升担当意识，主动迎接困难，并探索创新突破的路径，不得自己瞎编内容，不得出现具体数据，不得涉及高校工作外的内容。 -->
- 履行全面从严治党政治责任 <!-- 结合岗位职责，提出如何完善监督链条，压实责任，加强风险防控和教育引导，不得自己瞎编内容，不得出现具体数据，不得涉及高校工作外的内容。 -->
```

## 最终整合

```markdown
# Instruction
将初次撰写的读后感{{#1756724253021.text#}}与优化补充部分{{#1756726350157.text#}}整合，形成一篇完整、深入的最终版读后感。

## Role：党建读后感整合专家
你是一位党建写作专家，擅长对不同版本进行整合优化，确保逻辑连贯、思想深刻。

## Attention
1. 整合时要保证结构统一，避免重复。
2. 将相同章节的内容补充部分自然融入初稿。
3. 最终文本必须完整、流畅、可直接使用。
```



# 生成式提示词写法

工作流里有 **5 个 LLM 节点**

**（用户输入markdown模版格式 → 提示词生成器（针对初稿撰写）→ 初稿撰写 →提示词生成器（针对性优化） → 针对性优化 → 提示词生成器（针对最终整合）→ 最终整合）**

传入文件有 四个：

1、案例文件 （必填）

2、我的工作内容文件 （必填）

3、模版格式 （必填）

4、强化章节 （选填）



## 工作流步骤详解

> 先提示一下，dify 这样的工作流 通过html 的一些渲染，会导致 注释词 <!- XXXXXXX --> 在输出给用户看的时候是看不见的，但是实际上是存在的。而我们了 让用户看见 这些注释词，在代码块中用正则表达式处理了一下，不过实际上我保留了两个版本，用的还是原版的注释词 <!- XXXXXXX --> 输出，呈现给用户看单独搞了一个版本

### 获取案例文件和个人工作内容。

![623e55bc-5073-45fc-ad53-620d3b705e84](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/623e55bc-5073-45fc-ad53-620d3b705e84.png)

### 提取上传文件文字内容

![29f783f9-4fc7-4571-880e-a1941f0a854e](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/29f783f9-4fc7-4571-880e-a1941f0a854e.png)

### 提示词生成器（针对初稿撰写）

![e11f12c2-66c0-47ce-b1c6-3eddf6ffc9d0](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/e11f12c2-66c0-47ce-b1c6-3eddf6ffc9d0.png)

```markdown 
# Instruction
你将收到一份用户输入的 Markdown 大纲。  
你的任务是：**在保持大纲结构和文字完全不变的前提下**，在每个标题或小标题后补充一个 `<!-- -->` 注释。  

## 要求
- 不得改动用户输入的任何标题文字、层级、顺序。  
- 每一条标题或小标题都必须有对应的 `<!-- -->` 注释。   
- 注释内容要简洁、指令化，指导后续写作。  
- 注释内容必须与党建写作相关（如政治纪律、党性修养、责任担当、整改措施等）与案例{{#1756819593861.text#}}相关，与工作内容{{#1756819649172.text#}}尽可能关联上，并结合高校教职工的实际工作语境。  
- 输出仍然是 Markdown 格式。  

## Input
{{#1756805697432.template#}}

## OutputFormat
在用户输入的大纲基础上，逐条补充注释，示例如下：
带头严守政治纪律和政治规矩，维护党的团结统一
改为：带头严守政治纪律和政治规矩，维护党的团结统一<!-- 写出在政治纪律执行和信息传播管理方面存在的不足，结合高校岗位场景 -->
```

### 去除think部分

![80fb41af-50ab-43ca-af1e-b8d8a858beef](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/80fb41af-50ab-43ca-af1e-b8d8a858beef.png)

```python
import re

def main(prompt: str = "") -> dict:
    try:
        def remove_think(text: str) -> str:
            if not text:
                return ""
            return re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL)

        prompt_clean_think = remove_think(prompt)

        return {
            "result": "处理成功",
            "prompt_clean_think": prompt_clean_think.strip(),
            "error": ""
        }

    except Exception as e:
        # 出错时不要 raise，直接返回
        return {
            "result": "error",   # � 保证有输出字段
            "prompt_clean_think": "",
            "error": f"处理出错：{str(e)}，请重新输入"
        }

```

### 初稿撰写

![a5de9e06-e4de-4d27-8dba-f530de553314](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/a5de9e06-e4de-4d27-8dba-f530de553314.png)

```python
# Instruction
你将接收到一个已经补充了 `<!-- -->` 注释的 Markdown 大纲{{#1757401610434.prompt_clean_think#}}。  
请严格按照大纲的层级和顺序，展开撰写一篇规范的党建读后感文章。  

## Role：优秀党建学习示范者
你是一位认真学习党建文件的模范党员，能够结合自身工作与思想实际写出深刻的感悟。

## Attention
- 每个 `<!-- -->` 注释代表对应部分的写作指令，你必须完全吸收并落实其中的要求。  
- 最终输出中 **不能出现 `<!-- -->` 注释**，而是要将其转化为具体的内容。  
- 内容必须紧扣党建文件案例和高校教职工的实际工作语境，避免空泛套话。  
- 文章要有层次感，每个部分展开 2-4 段，既有思想剖析，也有具体措施。  
- 内容必须紧扣党建案例文件内容{{#1756819593861.text#}}，不能空泛。  
- 要结合本人高校教职工岗位特点{{#1756819649172.text#}}。 
- 章节标题与大纲中一模一样，一个字不改。

## OutputFormat
直接输出 Markdown 格式文章，结构必须与输入大纲{{#1757401610434.prompt_clean_think#}}完全保持一致，但正文内容要根据注释展开。  
```

### 去除think部分并调整注释格式

![4165685d-3ec3-4ea4-a35a-b45efa0d0098](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/4165685d-3ec3-4ea4-a35a-b45efa0d0098.png)

```python
import re

def main(arg1: str = "", arg2: str = "") -> dict:
    try:
        def remove_think(text: str) -> str:
            if not text:
                return ""
            return re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL)

        def unhide_comment(text: str) -> str:
            if not text:
                return ""
            # 用正则匹配 <!-- 任意内容 --> 并替换成 【内容】
            return re.sub(r'<!--\s*(.*?)\s*-->', r' 【注释：\1】', text)

        prompt_clean = unhide_comment(remove_think(arg1))
        # arg2_clean = unhide_comment(remove_think(arg2))
        # arg1_clean = remove_think(arg1)
        draft_clean = remove_think(arg2)

        # � 必须有至少一个 result 或自定义 key
        return {
            "result": "处理成功",
            "prompt_clean": prompt_clean.strip(),
            "draft_clean": draft_clean.strip(),
            "error": ""
        }

    except Exception as e:
        # 出错时不要 raise，直接返回
        return {
            "result": "error",   # � 保证有输出字段
            "prompt_clean": "",
            "draft_clean": "",
            "error": f"处理出错：{str(e)}，请重新输入"
        }

```

### 规范化输出样式

![3503456e-bed8-430c-b1c6-cf1c91843ca4](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/3503456e-bed8-430c-b1c6-cf1c91843ca4.png)

```markdown
># 生成的提示词
{{ prompt_clean }}

# ---------------------------

># 生成的初稿
{{ draft_clean }}
```

### 查看是否需要 针对性优化章节

![1a14791e-62a9-4f02-bf8b-e2cfc9ab0dd7](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/1a14791e-62a9-4f02-bf8b-e2cfc9ab0dd7.png)

### 输出 提示词和初稿

![c18eb1fa-61a2-4b59-adfd-af67669ec09b](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/c18eb1fa-61a2-4b59-adfd-af67669ec09b.png)

### 提示词生成器（针对性强化章节）

![0032cfef-ad43-488d-9bf8-ab73043fce3e](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/0032cfef-ad43-488d-9bf8-ab73043fce3e.png)

```markdown
# Instruction
你将收到一份用户输入的 Markdown 大纲。  
你的任务是：**在保持大纲结构和文字完全不变的前提下**，在每个标题或小标题后补充一个 `<!-- -->` 注释。  

## 要求
- 不得改动用户输入的任何标题文字、层级、顺序。  
- 每一条标题或小标题都必须有对应的 `<!-- -->` 注释。   
- 注释内容要简洁、指令化，指导后续写作。  
- 注释内容必须与党建写作相关（如政治纪律、党性修养、责任担当、整改措施等）与案例{{#1756819593861.text#}}相关，与工作内容{{#1756819649172.text#}}尽可能关联上，并结合高校教职工的实际工作语境。  
- 输出仍然是 Markdown 格式。  

## Input
{{#1756805697432.fortify_chapter#}}

## OutputFormat
在用户输入的大纲基础上，逐条补充注释，示例如下：
带头严守政治纪律和政治规矩，维护党的团结统一
改为：带头严守政治纪律和政治规矩，维护党的团结统一<!-- 写出在政治纪律执行和信息传播管理方面存在的不足，结合高校岗位场景 -->
```

### 去除think部分

![57a68497-c4d8-4a8c-9133-c122c4413fb8](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/57a68497-c4d8-4a8c-9133-c122c4413fb8.png)

```python 
import re

def main(prompt2: str = "") -> dict:
    try:
        def remove_think(text: str) -> str:
            if not text:
                return ""
            return re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL)

        prompt_clean_think2 = remove_think(prompt2)

        return {
            "result": "处理成功",
            "prompt_clean_think2": prompt_clean_think2.strip(),
            "error": ""
        }

    except Exception as e:
        # 出错时不要 raise，直接返回
        return {
            "result": "error",   # � 保证有输出字段
            "prompt_clean_think2": "",
            "error": f"处理出错：{str(e)}，请重新输入"
        }
```

### 针对性强化章节撰写

![微信图片_20250912092621_3561_17](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250912092621_3561_17.png)

```markdown
# Instruction
你将接收到一个已经补充了 `<!-- -->` 注释的 Markdown 大纲{{#17574056032950.prompt_clean_think2#}}。  
请严格按照大纲的层级和顺序，展开撰写一篇规范的党建读后感文章。  

## Role：优秀党建学习示范者
你是一位认真学习党建文件的模范党员，能够结合自身工作与思想实际写出深刻的感悟。

## Attention
- 每个 `<!-- -->` 注释代表对应部分的写作指令，你必须完全吸收并落实其中的要求。  
- 最终输出中 **不能出现 `<!-- -->` 注释**，而是要将其转化为具体的内容。  
- 内容必须紧扣党建文件案例和高校教职工的实际工作语境，避免空泛套话。  
- 文章要有层次感，每个部分展开 2-4 段，既有思想剖析，也有具体措施。  
- 内容必须紧扣党建案例文件内容{{#1756819593861.text#}}，不能空泛。  
- 要结合本人高校教职工岗位特点{{#1756819649172.text#}}。 
- 章节标题与大纲中一模一样，一个字不改。

## OutputFormat
直接输出 Markdown 格式文章，结构必须与输入大纲{{#17574056032950.prompt_clean_think2#}}完全保持一致，但正文内容要根据注释展开。  
```

### 去除think部分并调整注释格式

![fa7a1c16-0bd8-4967-a233-a0cc6bf05663](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/fa7a1c16-0bd8-4967-a233-a0cc6bf05663.png)

```python
import re

def main(arg1: str = "", arg2: str = "") -> dict:
    try:
        def remove_think(text: str) -> str:
            if not text:
                return ""
            return re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL)

        def unhide_comment(text: str) -> str:
            if not text:
                return ""
            # 用正则匹配 <!-- 任意内容 --> 并替换成 【内容】
            return re.sub(r'<!--\s*(.*?)\s*-->', r' 【注释：\1】', text)

        prompt_clean2 = unhide_comment(remove_think(arg1))
        # arg2_clean2 = unhide_comment(remove_think(arg2))
        # arg1_clean2 = remove_think(arg1)
        draft_clean2 = remove_think(arg2)

        # � 必须有至少一个 result 或自定义 key
        return {
            "result": "处理成功",
            "prompt_clean2": prompt_clean2.strip(),
            "draft_clean2": draft_clean2.strip(),
            "error": ""
        }

    except Exception as e:
        # 出错时不要 raise，直接返回
        return {
            "result": "error",   # � 保证有输出字段
            "prompt_clean2": "",
            "draft_clean2": "",
            "error": f"处理出错：{str(e)}，请重新输入"
        }
```

### 规范化输出样式

![4b32adc9-3b2b-4048-944b-0ec95d36d6c7](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/4b32adc9-3b2b-4048-944b-0ec95d36d6c7.png)

```markdown
># 生成的提示词
{{ prompt_clean2 }}

# ---------------------------
# ---------------------------

># 生成的初稿
{{ draft_clean2 }}
```

### 生成终稿

![8dff9d8e-f1ac-4de1-bb4f-dc423caf40b6](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/8dff9d8e-f1ac-4de1-bb4f-dc423caf40b6.png)

```markdown
# Instruction
将初次撰写的读后感{{#1757382286747.draft_clean#}}与优化补充部分{{#17574057721870.draft_clean2#}}整合，形成一篇完整、内容更加丰富的最终版读后感。

## Role：党建读后感整合专家
你是一位党建写作专家，擅长对不同版本进行整合优化，确保逻辑连贯、思想深刻。

## Attention
- 整合时要保证结构统一，避免重复。
- 将相同章节的内容补充部分自然融入初稿。
- 最终文本必须完整、流畅、可直接使用。
- 初次撰写的读后感{{#1757382286747.draft_clean#}}中不涉及{{#1756805697432.fortify_chapter#}}的章节内容和结构都不做修改。
- 将{{#17574057721870.draft_clean2#}}新生成的内容黏贴到初次撰写的读后感{{#1757382286747.draft_clean#}}对应章节下。

## OutputFormat
保持与{{#1756805697432.template#}}完全相同的结构，不用新增其他章节。
```

### 去除think部分

![83443527-1217-49d3-b7fa-22f4f1fd10de](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/83443527-1217-49d3-b7fa-22f4f1fd10de.png)

```python
import re

def main(arg1: str = "") -> dict:
    try:
        def remove_think(text: str) -> str:
            if not text:
                return ""
            return re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL)

        final_version_clean = remove_think(arg1)

        # � 必须有至少一个 result 或自定义 key
        return {
            "result": "处理成功",
            "final_version_clean": final_version_clean.strip(),
            "error": ""
        }

    except Exception as e:
        # 出错时不要 raise，直接返回
        return {
            "result": "error",   # � 保证有输出字段
            "final_version_clean": "",
            "error": f"处理出错：{str(e)}，请重新输入"
        }
```

### 规范化输出样式

![291536d2-dfe6-4af1-a30c-50928b34da67](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/291536d2-dfe6-4af1-a30c-50928b34da67.png)

```markdown
># 生成的初始提示词
{{ prompt_clean }}

# ---------------------------
# ---------------------------

># 生成的初稿
{{ draft_clean }}

# ---------------------------
# ---------------------------

># 生成的补强章节提示词
{{ prompt_clean2 }}

# ---------------------------
# ---------------------------

># 生成的补强章节内容
{{ draft_clean2 }}

# ---------------------------
# ---------------------------

># 生成的终稿
{{ final_version_clean }}
```

### 最终呈现

将 生成的初始提示词、生成的初稿、补强章节提示词、补强章节内容、终稿  呈现给用户。

![a8a8c8ae-2e87-47f4-b133-cf3c7290a55f](%E5%85%9A%E5%BB%BAagent%E7%BC%96%E5%86%99/a8a8c8ae-2e87-47f4-b133-cf3c7290a55f.png)