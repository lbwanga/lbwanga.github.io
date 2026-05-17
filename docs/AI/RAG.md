RAG 框架的四个基本组成部分：

- **输入（Input）**：即用户输入的问题，如果不使用 RAG，问题直接由大模型回答；
- **索引（Indexing）**：系统首先将相关的文档切分成段落，计算每个段落的 Embedding 向量并保存到向量库中；在进行查询时，用户问题也会以相似的方式计算 Embedding 向量；
- **检索（Retrieval）**：从向量库中找到和用户问题最相关的段落；
- **生成（Generation）**：将找到的文档段落与原始问题合并，作为大模型的上下文，令大模型生成回复，从而回答用户的问题；

https://www.aneasystone.com/archives/2024/06/advanced-rag-notes.html