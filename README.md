# 基于ChatGLM-6B的论文问答系统

本仓库实现了对于 ChatGLM-6B 模型基于 [P-Tuning v2](https://github.com/THUDM/P-tuning-v2) 的微调。实现了一个针对

## 软件依赖

首先需要在一台16GB显存，32G内存（够用就行）GPU云服务器上安装好显卡驱动cuda和pytorch框架，**下载ChatGLM源代码**之后先将服务启动起来，具体可以参见ChatGLM的README文档。也可以参见[卷福同学知乎博客](https://zhuanlan.zhihu.com/p/621216632)的分享。界面可以自己调控：

![image-20230606215220365](C:\Users\chenk\AppData\Roaming\Typora\typora-user-images\image-20230606215220365.png)

运行微调需要4.27.1版本的`transformers`。除 ChatGLM-6B 的依赖之外，还需要安装以下依赖

```
pip install rouge_chinese nltk jieba datasets
```

## 项目介绍

该项目对标ChatPaper，目的是通过ChatGPT实现对论文进行总结，帮助科研人进行论文初筛。项目目标总结为：

1. 实现本地化知识库检索与智能答案生成

2. 增加web search功能

3. 知识库选择功能和支持知识增量更新

项目技术路线为：

1. 向量知识库的构建：使用预训练大模型来完成。数据库格式可以有txt、json、xml，论文的切分方式可以参考chatpaper。
2. 针对GLM来进行微调，使用p-turning和Lora，参考文献较多。探究其中超参数的设置，比如参数rank，以及对比微调之后的效果等。(有精力的话研究源码)
3. Langchain框架的使用Langchain调用模型接口的修改，以及对接本地知识库的接口探究Langchain是否有高并发功能以及其他(如当网页关闭再次进入是否有问题<我有碰到过>)
4. 网络搜索，尝试爬虫
5. 探究Langchain及LLM对输入、回答的风险控制(符合法律法规、防止诱导问题等).
6. 基于GLM的可视化界面: 参考官方代码优化出基于本项目的可视化界面。

数据处理可以通过相应的Prompt命令从大语言模型中自动生成，部分代码：

```python
input_schema = ResponseSchema(name="input", description="the text inputed")
question_schema = ResponseSchema(name = "question", description="the question generated by llms")
answer_schema = ResponseSchema(name = "<ans>", description = "the anwser generated by lims")
response_schemas = [input_schema, question_schema, answer_schema]

output_parser = StructuredOutputParser.from_response_schemas(response_schemas)

format_instructions = output_parser.get_format_instructions()
print(format_instructions)

template_string = """Ask {numbers} and generate the answer base on the text \
that is delimited by triple backticks. \
text:```{text}```
{format instructions}
"""

prompt = PromptTemplate(
    input_variables=["numbers", "text", "format_instructions"],
    template=template_string,
)

llm =OpenAI()
llm_chain = LLMChain(prompt=prompt, llm=llm)
```

## 使用方法

由于我的代码还没搬运完，下面先搬运 [ADGEN](https://aclanthology.org/D19-1321.pdf) (广告生成) 介绍的代码使用方法。

### 下载数据集

ADGEN 数据集任务为根据输入（content）生成一段广告词（summary）。

```json
{
    "content": "类型#上衣*版型#宽松*版型#显瘦*图案#线条*衣样式#衬衫*衣袖型#泡泡袖*衣款式#抽绳",
    "summary": "这件衬衫的款式非常的宽松，利落的线条可以很好的隐藏身材上的小缺点，穿在身上有着很好的显瘦效果。领口装饰了一个可爱的抽绳，漂亮的绳结展现出了十足的个性，配合时尚的泡泡袖型，尽显女性甜美可爱的气息。"
}
```

从 [Google Drive](https://drive.google.com/file/d/13_vf0xRTQsyneRKdD1bZIr93vBGOczrk/view?usp=sharing) 或者 [Tsinghua Cloud](https://cloud.tsinghua.edu.cn/f/b3f119a008264b1cabd1/?dl=1) 下载处理好的 ADGEN 数据集，将解压后的 `AdvertiseGen` 目录放到本目录下。

### 训练

#### P-Tuning v2

运行以下指令进行训练：

```shell
bash train.sh
```

`train.sh` 中的 `PRE_SEQ_LEN` 和 `LR` 分别是 soft prompt 长度和训练的学习率，可以进行调节以取得最佳的效果。P-Tuning-v2 方法会冻结全部的模型参数，可通过调整 `quantization_bit` 来被原始模型的量化等级，不加此选项则为 FP16 精度加载。

在默认配置 `quantization_bit=4`、`per_device_train_batch_size=1`、`gradient_accumulation_steps=16` 下，INT4 的模型参数被冻结，一次训练迭代会以 1 的批处理大小进行 16 次累加的前后向传播，等效为 16 的总批处理大小，此时最低只需 6.7G 显存。若想在同等批处理大小下提升训练效率，可在二者乘积不变的情况下，加大 `per_device_train_batch_size` 的值，但也会带来更多的显存消耗，请根据实际情况酌情调整。训练过程如下：

![image-20230606211014966](C:\Users\chenk\AppData\Roaming\Typora\typora-user-images\image-20230606211014966.png)

如果你想要[从本地加载模型](../README_en.md#load-the-model-locally)，可以将 `train.sh` 中的 `THUDM/chatglm-6b` 改为你本地的模型路径。

#### Finetune

如果需要进行全参数的 Finetune，需要安装 [Deepspeed](https://github.com/microsoft/DeepSpeed)，然后运行以下指令：

```shell
bash ds_train_finetune.sh
```

## 引用

```
@inproceedings{liu2022p,
  title={P-tuning: Prompt tuning can be comparable to fine-tuning across scales and tasks},
  author={Liu, Xiao and Ji, Kaixuan and Fu, Yicheng and Tam, Weng and Du, Zhengxiao and Yang, Zhilin and Tang, Jie},
  booktitle={Proceedings of the 60th Annual Meeting of the Association for Computational Linguistics (Volume 2: Short Papers)},
  pages={61--68},
  year={2022}
}
```
