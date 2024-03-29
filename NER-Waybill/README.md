# 快递单信息抽取

### 一、项目说明

在该项目中，主要跟大家介绍如何使用预训练语言模型实现对快递单信息的抽取，从用户提供的快递单中，抽取姓名、电话、省、市、区、详细地址等内容，形成结构化信息。辅助物流行业从业者进行有效信息的提取，从而降低客户填单的成本。

### 二、快递单信息抽取

#### 任务介绍

如何从物流信息中抽取想要的关键信息呢？我们首先要定义好需要抽取哪些字段。

比如现在拿到一个快递单，可以作为我们的模型输入，例如“张三18625584663广东省深圳市南山区学府路东百度国际大厦”，那么模型的目的就是识别出其中的“张三”为人名（用符号 P 表示），“18625584663”为电话名（用符号 T 表示），“广东省深圳市南山区百度国际大厦”分别是 1-4 级的地址（分别用 A1~A4 表示，可以释义为省、市、区、街道）。

这是一个典型的命名实体识别（Named Entity Recognition，NER）场景，各实体类型及相应符号表示见下表：

| 抽取实体/字段 | 符号 | 抽取结果     |
| :------------ | :--- | :----------- |
| 姓名          | P    | 张三         |
| 电话          | T    | 18625584663  |
| 省            | A1   | 广东省       |
| 市            | A2   | 深圳市       |
| 区            | A3   | 南山区       |
| 详细地址      | A4   | 百度国际大厦 |

#### 模型介绍

我们可以用序列标注模型来解决快递单的信息抽取任务，下面具体介绍一下序列标注模型。

在序列标注任务中，一般会定义一个标签集合，来表示所以可能取到的预测结果。在本案例中，针对需要被抽取的“姓名、电话、省、市、区、详细地址”等实体，标签集合可以定义为：

label = {P-B, P-I, T-B, T-I, A1-B, A1-I, A2-B, A2-I, A3-B, A3-I, A4-B, A4-I, O}

每个标签的定义分别为：

| 标签 | 定义                       |
| :--- | :------------------------- |
| P-B  | 姓名起始位置               |
| P-I  | 姓名中间位置或结束位置     |
| T-B  | 电话起始位置               |
| T-I  | 电话中间位置或结束位置     |
| A1-B | 省份起始位置               |
| A1-I | 省份中间位置或结束位置     |
| A2-B | 城市起始位置               |
| A2-I | 城市中间位置或结束位置     |
| A3-B | 县区起始位置               |
| A3-I | 县区中间位置或结束位置     |
| A4-B | 详细地址起始位置           |
| A4-I | 详细地址中间位置或结束位置 |
| O    | 无关字符                   |

注意每个标签的结果只有 B、I、O 三种，这种标签的定义方式叫做 BIO 体系，也有稍麻烦一点的 BIESO 体系，这里不做展开。其中 B 表示一个标签类别的开头，比如 P-B 指的是姓名的开头；相应的，I 表示一个标签的延续。

对于句子“张三18625584663广东省深圳市南山区百度国际大厦”，每个汉字及对应标签为：

![img](https://ai-studio-static-online.cdn.bcebos.com/1f716a6ad48649cc99c56c27108773bea6b0afa3f36e4efba4851641658b2414)

​                                                                                 数据集标注示例

注意到“张“，”三”在这里表示成了“P-B” 和 “P-I”，“P-B”和“P-I”合并成“P” 这个标签。这样重新组合后可以得到以下信息抽取结果，实现对每个有用字段的抽取。

| 张三 | 18625584663 | 广东省 | 深圳市 | 南山区 | 百度国际大厦 |
| :--: | :---------: | :----: | :----: | :----: | :----------: |
|  P   |      T      |   A1   |   A2   |   A3   |      A4      |

#### 项目难点

（1） 预处理来提高数据质量。
实际应用数据中可能包含大量噪声。为了降低其对模型的影响，需要替换或者删除掉特殊字符以匹配现有的词典。值得注意的是，词典的质量也可能影响模型最终的标注效果。

（2）设计序列标注模型时需要考虑地址的层级结构。
完整的地址信息一般包含了省、市、区（县）、街道等不同层次的信息。序列建模时要同时考虑到文本中不同词之间的层次结构关系，来保证模型设计的合理性和有效性。ERNIE中的attention机制可以在一定程度上表示这一结构，除此之外，还可以进一步使用CRF等方法进行显式的建模。

（3）微调过程中的学习率设置。
预训练模型为序列标注任务提供了丰富的语法信息，在微调过程中需要注意学习率的设置。warmup策略初期较大的学习率可以加速模型的收敛。学习率随着模型的迭代不断衰减，逐步收敛得到一个适用于命名实体识别任务的模型。

#### 数据准备

我们使用的是已标注的快递单关键信息数据集。训练使用的数据也可以由大家自己组织数据。数据格式除了第一行是 `text_a\tlabel` 固定的开头，后面的每行数据都是由两列组成，以制表符分隔，第一列是 utf-8 编码的中文文本，以 `\002` 分割，第二列是对应每个字的标注，以 `\002` 分割。

在训练和预测阶段，我们都需要进行原始数据的预处理，具体处理工作包括：

1. 从原始数据文件中抽取出句子和标签，构造句子序列和标签序列
2. 将句子序列中的特殊字符进行转换
3. 依据词典获取词对应的id索引

#### 模型训练

PaddleNLP提供了丰富的模型，在本项目中我们主要采用Bi-GRU，ERNIE，ERNIE-Gram，ERNIE+CRF模型对快递单信息进行抽取，并将不同模型的快递单抽取性能进行对比，为广大用户提供参考。

运行项目：

##### Bi-GRU抽取

```python
python run_bi_gru_crf.py
```

##### ERNIE抽取

```python
python run_ernie.py
```

##### ERNIE-Gram抽取

```
python run_ernie_gram.py
```

##### ERNIE+CRF抽取

```python
python run_ernie_crf.py
```

#### 模型效果对比

|    模型    | F1(%) |
| :--------: | :---: |
| Bi-GRU+CRF | 97.7  |
|   ERNIE    | 98.6  |
| ERNIE+CRF  | 97.3  |
| ERNIE-Gram | 99.3  |

由上表可以看出，预训练语言模型的性能要高于采用传统的Bi-GRU+CRF，其中ERNIE-Gram的效果最好，其F1值达到99.3%，预测结果会保存在相应的result.txt文件中。



























































