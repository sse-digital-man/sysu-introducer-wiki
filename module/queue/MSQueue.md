# 基于语义分级的先入先出动态消息队列（MSQueue）

核心思想是：通过将消息按其**消息类型**和**语义相关性**进行分类管理。

实现方法包括三个队列：

* `queue1` 用于存储管理员消息
* `queue2` 和 `queue3` 分别存储与主题高度相关和不相关的用户消息。这里使用`Vector Similarity`计算语义相似度分数将弹幕消息分为（0\~0.4相关，0.4\~1不相关）两类

我们在各个消息队列中使用**先入先出**的策略。即按照 `queue1`、`queue2`、`queue3` 的顺序依次弹出，实现了管理员消息优先处理和语义相关性较高消息的优先级管理。

```
class MSQueue(DynamicMessageQueue):
    def push(self, message: Message):
    // 省略

    def pop(self) -> Message:
        if len(self.queue1) != 0 :
            return self.queue1.pop(0)
        if len(self.queue2) != 0 :
            return self.queue2.pop(0)
        if len(self.queue3) != 0 :
            return self.queue3.pop(0)
        raise ValueError("the queue is empty")
```
