# 大语言模型

大语言模型模块是数字人系统的核心组件，它充当数字人的大脑，负责回答消息。

值得注意的是，本模块并不负责对接受的消息进行筛选，也就是说该模块会回答接收到的所有消息，当然该模块也会对不同的消息使用不同的回答策略。

## 概述

### 现有技术不足分析

我们来看 `GPT-3.5` 本身的回答能力：


| Role      | Content                                            |
| --------- | -------------------------------------------------- |
| User      | 中山大学的校训是什么？                             |
| Assistant | 中山大学的校训是："自强不息，厚德载物"。           |
| Truth     | 中山大学的校训是："博学、审问、慎思、明辨、笃行"。 |

显然，`GPT-3.5` 并不了解中山大学的校训是什么，甚至连中山大学他都可能认为在中山。
倘若用 `GPT-4o` 或者其他更先进的模型或者具有联网功能的模型，效果会有所提升。
但考虑到 API 的成本问题、以及或许可能接入更多本地国产大模型，知识的深度与广度可能会达不到我们的要求。

### 优化方向

大语言模型的能力可以由两方面衡量：

1. 指令遵循能力：大模型需要怎么表现？比如面对选择题时，应该回答一个选项 A 还是回答选项并解释？
2. 知识能力：大模型了解哪些知识？如何让大模型了解特定领域的知识，比如说中大的校史。

对于提升指令遵循能力，我们可以通过**提示工程**技术实现，通过编写合适的 `prompt`，让大模型理解任务。
而对于如何提升大模型的知识能力，也就是回答垂直领域的能力，是本项目的关键。

## 模块结构

本模块既需要使用大语言模型的文本生成能力，又需要检索数据库来获取专业知识，而且这两项工作会又多种实现。因此本系统将该模块拆分成两个子模块 `caller` 和 `searcher`，将调用 LLM 的逻辑和知识检索逻辑分别封装在这两个子模块中，而本模块则负责实现利用这两个子模块实现回答问题的能力。

### 1. 调用器 Caller

目前本项目仅支持如下几种 LLM 的 API 的调用方式，
并可以通过配置文件进行动态加载和配置。

#### (1) GPT

使用 openai sdk 进行调用 GPT3.5。

- `apiKey`: 调用 API 所需要的秘钥
- `url`: openai 代理的链接。具体使用说明可以参考 [GPT_API_free](http://github.com/chatanywhere/GPT_API_free)

> TODO: 描述 `prompt`

#### (2) Virtual

如果使用 LLM 模块进行生成文本，会在使用过程中产生相应的费用。
因此，如果在项目开发时，并不特别需要大模型的文本生成的能力，
可以使用此处的 `VirtualBot`，以减少开发调试过程中产生不必要的 token 开销。

该部分提供两种生成回答的策略。第一种随机策略，从随机回答库中选择一条文本进行输出。
第二种是复读策略，会直接输出“我回答了 XXX”。

- `delay`: 用于控制延迟
- `isRandom`: 选择生成答案的策略

### 2. 检索器 Searcher

#### (1) ElasticSearch

`Elasticsearch`是一个开源的实时分布式搜索和分析引擎，它建立在Apache Lucene库之上，被设计用于处理大规模数据集，具有高性能、可伸缩性和强大的全文搜索功能。

工作原理：通过倒排索引`inverted index`机制，建立起文档中的每个词映射到包含该词的文档的索引，`Elasticsearch`能够快速定位包含特定词汇的文档，并根据相关性评分进行排序，从而实现高性能的全文搜索功能。

我们使用ElasticSearch主要完成了下面的任务：

* 索引创建：将中大语料库数据对象，存储在Elasticsearch中。在索引中，每个文档都是一个JSON格式的数据对象，包含了要检索的数据。可以根据数据模型设计文档的结构，将数据库中的字段映射到文档的字段。每个文档都会被分配一个唯一的ID，用于在搜索和检索时进行引用。
* 关键词搜索：一旦数据被索引到Elasticsearch中，就可以通过发送搜索查询来执行基于关键词的检索。查询可以包含一个或多个关键词、条件和过滤器，以精确匹配所需的数据。Elasticsearch会根据查询条件在倒排索引中查找匹配的文档，并返回与查询相匹配的结果。

与下面的词向量嵌入方法相比，ElasticSearch优势在于：

1. **速度快**，Elasticsearch可以更快地定位和检索包含关键词的文档。这使得它能够在很短的时间内返回查询结果，平均查询时间可以达到0.2秒左右。
2. **占用内存少**，ElasticSearch以较少的内存占用来支持快速的搜索和检索操作
3. 基于大模型提取关键词的前置条件下，可以在得到不俗的检索结果的同时，检索用时大大减少。

#### (2) Vector Similarity

`Vector Similarity` 是稠密检索的主要思路。
基于 `word embedding` 的技术，通过计算向量之间的相似度来实现检索。

实现上，我们采用 `langchain_openai` 的 `OpenAIEmbeddings`，
背后是一个训练用于完成 `word embedding` 的 `Transformer` 结构。
同时，向量数据库我们采用 `langchain_community.vectorstores` 的 `Chroma`。

整体的流程如下：

1. 将数据库中的每一条数据，使用 `OpenAIEmbeddings` 嵌入到向量空间
2. 将向量空间的数据组织存储到 `Chroma`
3. `query` 到来之后，同样嵌入到向量空间
4. 使用 `Chroma` 进行 `query` 和数据之间的相似度检索，选取 Top-K 的结果
5. 使用阈值过滤相似度距离高于阈值的结果

在其中，存在两个技术细节：

**1 嵌入模板**

嵌入模板对基于 `Transformer` 的嵌入模型的效果有着巨大的影响。
这是由语言模型的特质所决定的，离散的词对嵌入的结果的影响是非常巨大的，在部分论文  [Making Pre-trained Language Models Better Few-shot Learners](https://arxiv.org/abs/2012.15723) 中也有提及。

一个好的模板，应该能够显著提升 `Transformer` 模型在特定任务上的表现。
由于我们是一个检索任务，且与中大校史相关，所以我设计了如下的模板，来连接数据。

```python
self.__prompt_template = """
请回答用户关于中山大学信息的查询
查询: {query}
回答: {answer}
"""
data = self.__prompt_template.format(query=value['query'], answer=value['document'])
```

效果如下：

```
[query] 
中山大学的饭堂什么水平

[w/o templete]
1. 中山大学的校名...
2. 中山大学的办学层次...
3. 中山大学的校训...

[w/ templete]
1. 广州东校区学四食堂饮食...
2. 广州东校区行政楼餐厅饮食...
3. 珠海校区榕园食堂饮食...
```

**2 阈值**

收集了开发交流平台的广大开发者的阈值设置，针对 `text-embedding-ada-002` 的阈值设置一般在 `0.3~0.4` 之间。我们进行了少样本人工评估，最终选定阈值为 `0.35`。
