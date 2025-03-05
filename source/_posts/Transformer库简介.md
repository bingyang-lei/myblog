---
title: Transformer库简介
date: 2025-03-01 23:50:51
tags: AI
---
笔者最近对AI agent相关内容较感兴趣，因此对Transformer库进行了简单的了解。

Hugging Face Transformers 库的模块设计非常结构化，不同子类负责不同功能。

以下是核心模块的作用和典型应用场景的详细说明：（注：内容来自AI生成+个人修改）

---

### 一、**模型核心类**（构建和加载模型）

#### 1. **`PreTrainedModel`**（基类）
- **作用**：所有模型的抽象基类，定义了模型加载/保存、权重共享等通用方法。
- **特点**：
  - 不直接使用，通过子类（如 `BertModel`, `GPT2LMHeadModel`）实例化。
  - 提供 `from_pretrained()` 方法（核心API）。
- **示例**：
  ```python
  from transformers import PreTrainedModel
  # 实际使用时需继承并实现 forward 方法（自定义模型）
  class MyModel(PreTrainedModel):
      def __init__(self, config):
          super().__init__(config)
          # 自定义层定义
  ```

---

#### 2. **`AutoModel` 系列**（自动推断模型）
- **作用**：根据模型名称或路径自动推断并加载预训练模型。
- **常见子类**：
  | 类名                                 | 适用场景                |
  | ------------------------------------ | ----------------------- |
  | `AutoModel`                          | 通用模型（无任务头）    |
  | `AutoModelForCausalLM`               | 文本生成（如 GPT）      |
  | `AutoModelForSequenceClassification` | 文本分类（如 BERT）     |
  | `AutoModelForQuestionAnswering`      | 问答任务（如 RoBERTa）  |
  | `AutoModelForMaskedLM`               | 掩码语言建模（如 BERT） |
  | `AutoModelForTokenClassification`    | 序列标注（如 NER）      |

- **示例**：
  ```python
  from transformers import AutoModelForCausalLM
  # 自动加载生成类模型（如 GPT-2）
  model = AutoModelForCausalLM.from_pretrained("gpt2")
  ```

---

### 二、**分词器（Tokenizer）**
#### `AutoTokenizer`
- **作用**：将文本转换为模型可处理的输入（如 token IDs、attention_mask）。
- **关键功能**：
  - 分词（`tokenize`）、编码（`encode`）、批量处理（`__call__`）。
  - 处理特殊标记（如 `[CLS]`, `[SEP]`）。
- **示例**：
  ```python
  from transformers import AutoTokenizer
  tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
  inputs = tokenizer("Hello world!", return_tensors="pt")
  ```

---

### 三、**配置类（Config）**
#### `PretrainedConfig`
- **作用**：存储模型结构参数（如层数、隐藏层维度），控制模型构建。
- **使用场景**：
  - 修改预训练模型的架构（如调整 `hidden_size`）。
  - 从头训练新模型时定义配置。
- **示例**：
  ```python
  from transformers import BertConfig, BertModel
  # 自定义配置
  config = BertConfig(hidden_size=768, num_attention_heads=12)
  model = BertModel(config)  # 根据配置初始化模型
  ```

---

### 四、**管道（Pipeline）**
#### `pipeline`
- **作用**：快速调用预训练模型完成端到端任务（封装模型+分词器+后处理）。
- **支持任务**：
  ```python
  # 常见任务类型
  pipeline("text-generation")          # 文本生成
  pipeline("text-classification")      # 分类
  pipeline("question-answering")       # 问答
  pipeline("translation_en_to_fr")     # 翻译
  ```
- **示例**：
  ```python
  from transformers import pipeline
  classifier = pipeline("sentiment-analysis")
  print(classifier("I love Transformers!"))  # 输出: [{'label': 'POSITIVE', 'score': 0.9998}]
  ```

---

### 五、**训练工具**
#### 1. **`Trainer` 类**
- **作用**：封装训练循环（优化器、学习率调度、分布式训练等）。
- **依赖数据集格式**：需配合 `Dataset` 或 `torch.utils.data.Dataset` 使用。
- **示例**：
  ```python
  from transformers import Trainer, TrainingArguments
  training_args = TrainingArguments(output_dir="./results", num_train_epochs=3)
  trainer = Trainer(
      model=model,
      args=training_args,
      train_dataset=train_dataset,
  )
  trainer.train()
  ```

#### 2. **`TrainingArguments`**
- **作用**：定义训练超参数（如批次大小、学习率、保存策略）。
- **关键参数**：
  ```python
  TrainingArguments(
      output_dir="output",
      learning_rate=2e-5,
      per_device_train_batch_size=16,
      logging_steps=100,
      save_steps=500,
  )
  ```

---

### 六、**其他实用工具**
#### 1. **`AutoFeatureExtractor`**
- **作用**：处理非文本输入（如音频、图像）。
- **示例**：
  ```python
  from transformers import AutoFeatureExtractor
  extractor = AutoFeatureExtractor.from_pretrained("facebook/wav2vec2-base")
  ```

#### 2. **`AutoProcessor`**
- **作用**：多模态任务中统一处理文本+非文本输入（如 CLIP）。
- **示例**：
  ```python
  from transformers import AutoProcessor
  processor = AutoProcessor.from_pretrained("openai/clip-vit-base-patch32")
  ```

---

### 七、**模块协作关系图**
```
文本输入 → AutoTokenizer → 编码 → AutoModelForXXX → 输出 logits
                              ↓
配置控制 → PretrainedConfig → 影响模型结构
                              ↓
训练流程 → Trainer + TrainingArguments → 微调模型
```

---

### 八、**最佳实践总结**
1. **基础使用**：
   ```python
   # 加载模型+分词器
   model = AutoModelForSequenceClassification.from_pretrained("bert-base-uncased")
   tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
   ```
   
2. **自定义配置**：
   ```python
   config = BertConfig(hidden_size=1024, num_labels=5)
   model = BertModel(config)
   ```

3. **快速推理**：
   ```python
   pipe = pipeline("text-generation", model="gpt2")
   print(pipe("The future of AI is"))
   ```

4. **微调训练**：
   ```python
   trainer = Trainer(
       model=model,
       args=TrainingArguments(...),
       train_dataset=dataset,
   )
   trainer.train()
   ```

通过灵活组合这些模块，可以实现从快速实验到工业级部署的全流程 NLP 任务处理。