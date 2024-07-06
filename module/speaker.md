# 文本转语音（TTS）

文本转语音模块在数字人系统与观众的交互中至关重要，它赋予数字人自然流畅的语音表达，使交流更加生动亲切，传递情感和意图，提升用户互动体验，真正实现人与机器间的无缝沟通。

目前主流的文本转语音实现方案有两种：一种是使用 TTS 服务提供商的 API，如： [Azure-tts](https://learn.microsoft.com/zh-cn/azure/ai-services/speech-service/index-text-to-speech)、[Openai-tts](https://platform.openai.com/docs/guides/text-to-speech)等，一种是使用开源本地模型，如：
[Bert-VITS2](https://github.com/fishaudio/Bert-VITS2)、
[GPT-SoVITS](https://github.com/RVC-Boss/GPT-SoVITS)、[ChatTTS](https://github.com/2noise/ChatTTS?tab=readme-ov-file)等。

为方便后续使用不同的实现方法，我们为文本转语音模块定义了几个常用的接口，后续方案只需通过实现接口便可以运行。

## 接口

在本模块中，主要通过不同类型的对象来提供文本转语音的功能。
其中，它具有的接口如下：

```python
# 抽象方法，不同类型的对象实现该方法以生成语音
@abstractmethod
def speak(self, text) -> str:
    ...

# 根据类型名，生成格式化的音频文件名
def _generate_filename(self, suffix: str = "wav") -> str:
    ...
```

通过上述接口我们已经可以实现文本转语音的各种基本功能
若希望额外实现功能，也可以通过在子类添加参数来实现重写，并在调用时加以区分。

## 实现方案

目前，本项目实现了 Azure-tts、Bert-VITS2、GPT-SoVITS 这三种方案，音频示例可查看 [此处](https://github.com/sse-digital-man/sysu-introducer-wiki/blob/main/module/audios)。

### Azure-tts

[Azure-tts](https://learn.microsoft.com/zh-cn/azure/ai-services/speech-service/index-text-to-speech)是微软 Azure 提供的使用深度神经网络训练得到的模型，我们可以使用其提供的 SDK 来合成语音。其提供的服务十分灵活，可以使用语音合成标记语言 (SSML，一种基于 XML 的标记语言) 微调文本转语音输出属性，例如音调、发音、语速、音量等。与纯文本输入相比，它可以提供更多的控制权和灵活性。

我们制作了 ssml 风格模版，并且可通过`_prepare_ssml(self, text: str) -> str`定制化输出语音

```xml
<speak
    xmlns="http://www.w3.org/2001/10/synthesis"
    xmlns:mstts="https://www.w3.org/2001/mstts"
    version="1.0" xml:lang="zh-CN"
>
    <voice name="zh-CN-XiaoyiNeural">
        <mstts:express-as style="hopeful" styledegree="2">
            {text}
        </mstts:express-as>
    </voice>
</speak>
```

<audio controls>
    <source src="audios/Azure-TTS.wav" type="audio/wav">
</audio>

### Bert-VITS2

[Bert-VITS2](https://github.com/fishaudio/Bert-VITS2) 结合了两种关键技术：[BERT](https://en.wikipedia.org/wiki/BERT_%28language_model%29)（基于 Google 提出的 Transformer 模型的预训练语言表示模型）和 VITS（基于变分推断的转换器结构）。它通过利用预训练的 BERT 模型从文本中提取高质量的语言表示，并结合 VITS 模型将这些表示转换为自然流畅的语音输出。

我们根据中大校史介绍官的人设，基于[AI-Hobbyist/Genshin_Datasets: Genshin Datasets For SVC/SVS/TTS](https://github.com/AI-Hobbyist/Genshin_Datasets)数据集，对数据进行一定的预处理（重采样、切分等等）后训练了对应的模型。

同时为方便部署，提供 WebAPI 调用的 Docker Image: [kingkia/bert-vits2-api - Docker Image | Docker Hub](https://hub.docker.com/r/kingkia/bert-vits2-api)

<audio controls>
    <source src="audios/Bert-VITS2.wav" type="audio/wav">
</audio>

### GPT-SoVITS

[GPT-SoVITS](https://github.com/RVC-Boss/GPT-SoVITS)基于[GPT](https://en.wikipedia.org/wiki/Generative_pre-trained_transformer)（基于 Transformer 模型的预训练语言模型）和 SoVITS（基于变分推断的转换器结构）。其通过预训练的 GPT 模型生成文本表示，并利用 SoVITS 模型将这些文本表示转换为高质量的语音输出。

为方便部署，提供 WebAPI 调用的 Docker Image: [kingkia/gpt-sovits-api - Docker Image | Docker Hub](https://hub.docker.com/r/kingkia/gpt-sovits-api)

<audio controls>
    <source src="audios/GPT-SoVITS.wav" type="audio/wav">
</audio>
