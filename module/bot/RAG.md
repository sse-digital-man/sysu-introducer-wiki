# RAG

本实现类型采用 `RAG(Retrieval Augmented Generation)` 检索增强生成技术，构建了相关的知识数据库，再与**检索器**，**提示工程**等技术结合，来提升大模型的知识能力。

## 处理逻辑

### 1. 检索增强生成 Rag

检索增强生成的流程可以直观地分为：1)检索，2)增强，3)生成。代码框架如下：

```python
class RagBot(BasicBot):
    demonstrate_prompt = ...
    data_prompt = ...
    def load_config(self):
        ...
    def retrieve_sim_k(self, query: str, k: int) -> Dict[str, str]:
        ...
    @BasicModule._handle_log
    def talk(self, query: str) -> str:
        ...
```

#### 检索

想要完成检索任务，首先我们需要有相对应的数据。

> TODO: 添加数据库收集的总结

有了数据，便可以开始检索工作。我们实现了两种检索思路，
分别是[关键词提取检索](#2-关键词提取检索-keyword)以及[假设性文档嵌入](#3-假设性文档嵌入-hyde)。

#### 增强

为了将检索的相关信息用于增强大模型能力，我们利用大模型强大的 `上下文学习(In-context Learning)` 能力，通过**提示词工程**来实现。
因此，我们定义了如下的 `prompt` 模板：

```python
system_prompt = \
"""### 示范一
[用户问题]
国际学生政策是怎样的？
[参考资料]
参考资料1:
标题:国际学生政策
内容:详情可见中山大学留学生办公室官网
[回答]
国际学生政策，官网有最权威的信息哦！快去中山大学留学生办公室官网看看吧，你会有新发现的！

### 示范二
[用户问题]
可以介绍一下你自己嘛？
[参考资料]
参考资料1:
标题:中山大学校校歌歌词
内容:白云山高,珠江水长,吾校矗立,蔚为国光,中山手创
[回答]
嗨！我是中小大，中大软件工程学院的大三学生，也是中大介绍官。我超爱写代码和阅读历史书籍！

### 示范三
[用户问题]
为什么皮卡丘喜欢放电
[参考资料]
参考资料1:
标题:中山大学的微电子科学与技术学院本科招生专业
内容:微电子科学与工程
参考资料2:
标题:中山大学的集成电路学院本科招生专业
内容:微电子科学与工程
[回答]
很抱歉，我是中大介绍官，不能回答关于皮卡丘的问题哦~

### 开始任务
[用户问题]
{query}
[参考资料]
{data_str}
[回答]"""

data_prompt = \
"""参考资料{i}:
标题:{query}
内容:{document}
"""
```

模板包含两个部分：

**示范(demonstration)：** 通过 `Few-shot Learning`，增强大模型的指令遵循能力。
我们提供了三个范例，分别对应三种情况：

-   问题与检索内容相关，可以根据问题与检索内容回答。
-   问题相关，检索得到的内容不相关，只根据用户问题。
-   用户问题与中大无关，不应该回答。

**检索增强内容：** 以 `[参考资料]` 的形式给到模型。

#### 生成

将最终融合的 `prompt` 给到[调用器 Caller](#调用器-caller) 生成答案。

### 2. 关键词提取检索 Keyword

> TODO: 如何使用两个子模块生成消息的回答...

### 3. 假设性文档嵌入 HyDE

`Hyde`(Hypothetical Document Embeddings，假设性文档嵌入)通过使用一个大语言模型，在响应查询的时候建立一个假设的文档。
通过计算假设文档的向量而在[向量数据库](#2-vector-similarity)中搜索。
该方法来源于论文 [Precise Zero-Shot Dense Retrieval without Relevance Labels](https://arxiv.org/abs/2212.10496)。

`HyDE` 考虑到的是在 `查询-回答` 任务中，`查询` 与 `回答` 的相似度可能不高，不如生成一个假设的回答，从而通过这个假设的回答在向量数据库中进行检索。

代码实现上也非常简单：

```python
def retrieve_sim_k(self, query: str, k: int) -> Dict[str, str]:
    # 基本的检查...

    # 1. 生成 hyde 假设性回答
    query = searcher.prompt_template.format(query=query)
    query += caller.single_call(query, False)

    # 2.使用 hyde 检索得到 top-k 相似结果
    retrieve_res = searcher.search_with_label(query, k)

    return retrieve_res
```
