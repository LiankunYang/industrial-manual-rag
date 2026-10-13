# 工业产品手册 RAG 问答原型

基于施耐德 Easy UPS 公开中文手册的个人复现实验。目标是从手册中检索依据、回答问题并保留可核对的页码引用，重点观察漏检、错误前提和资料缺失。项目与厂商没有官方合作关系。

## 目标与当前范围

当前使用一份22页手册中的7页：PDF第4、5、6、11、14、15、16页，生成26个文档块。范围包括规格、软件通信、显示设置和事件日志，覆盖的是选定页面。

程序、问题集、历史结果和带输出的Notebook集中在 `public_manual/`，保持原项目结构。该展示版本基于已有练习程序完成复现、结果核对和漏检诊断。

## 问答流程

**公开PDF → 文字提取 → 保留页码的短块 → BGE检索 → 原问题与参考原文 → 带引用的回答**

- [资料准备与基线问答](public_manual/manual_project.py)：使用pypdf解析；每块最多260个字符，相邻块重叠60个字符，不跨页，保留PDF物理页码和块编号。
- BGE `bge-small-zh-v1.5` 将原文编码为512维归一化向量，问题编码后按余弦相似度检索。原文向量准备一次，后续问题复用。
- DeepSeek接收原问题与候选原文；提示词要求依据资料回答并标明来源，摘录没有依据时承认资料不足。引用和回答仍需人工核对。
- 结果记录包含问题、预期、候选原文、页码、实际回答和人工评价。预期答案仅供事后检查，不发送给模型。

基线的运行与实际输出见 [demo.ipynb](public_manual/demo.ipynb)。

## 检索改进

[混合检索程序](public_manual/retrieval_upgrade.py) 在语义检索之外增加字符TF-IDF：

1. 对匹配文本去掉空白，缓解PDF把“高效率”等词拆行的问题，原始文字和引用保留。
2. 使用2–4个字符片段建立TF-IDF索引，与语义检索分别取最多8个候选。
3. 用RRF融合两路排名，单路贡献为 `1 / (60 + rank)`，最终仍取3块。

没有用题号、预期答案、指定页码或人工关键词表选择候选。运行和对比见 [hybrid.ipynb](public_manual/hybrid.ipynb)。融合分数用于排序，不等同于语义相似度或回答正确概率。

## 评测结果与失败案例

各轮独立记录，不合并为系统整体准确率。`status=ok`表示请求完成，`reference_page_hit`表示命中预期页，都不能自动判定答案通过。

| 轮次 | 问题范围 | 已有证据支持的结果 | 记录 |
|---|---|---|---|
| 语义检索基线 | 15道固定题 | 单次人工核对摘要为12道回答或拒答符合预期；R03、R04、R09漏答。原JSON中R01、R05另有显式人工评价。 | [基线15题](public_manual/results/llm07_all15.json) |
| 混合检索改进 | 同一15道固定题，参与了调试 | R03、R04、R09已有通过的人工评价。其余题没有逐条新增评价，未将这轮描述为15题全部通过。 | [当轮对比基线](public_manual/results/llm08_baseline15.json)、[混合检索15题](public_manual/results/llm08_all15.json) |
| 新问题检查 | 5道未参与此前调试的新题 | 单次人工核对4/5通过；U03仍因漏检未通过。这是一次小样本检查。 | [五题和标准](public_manual/unseen_cases09.json)、[首次结果与五条评价](public_manual/results/llm09_new5_20261013_121225_300020.json) |
| 人工查询表达诊断 | U03已进入调试 | 人工调整检索表达后找到了依据，单例人工评价通过；首次新题检查仍保留4/5。 | [诊断程序](public_manual/query_diagnosis.py)、[诊断结果](public_manual/results/llm10_query_diagnosis_20261013_131350_959665.json) |

代表性失败及定位：

- **R03、R04**：候选命中了第15页，但没有包含默认设置表所在的块01。混合检索将该块加入候选，回答分别给出报警启用、绿色模式禁用。
- **R09**：候选命中了第16页块02，却没有含日志数量的块01。混合检索补进块01，回答给出最后10个事件。
- **U03**：首次候选为 `p06-c02`、`p15-c02`、`p06-c01`，没有第4页的贮存充电条款。人工检索表达“电池贮存期间的充电要求和充电周期”找到了 `p04-c02`，新回答给出每六个月充电一次。这里支持人工调整检索表达，没有实现自动查询改写。

细节与已有输出见 [新问题检查](public_manual/new_questions.ipynb) 和 [查询表达诊断](public_manual/query_diagnosis.ipynb)。第一次检查和后续调试结果分别保存，诊断成功不改写原成绩。

[取1块与3块的对比](public_manual/results/llm07_topk_comparison.json) 中，PowerChute问题两组回答均符合依据，参考文字由250字符增至872字符；这一单例没有显示回答改善。

## 安装与运行

使用Python 3.10及以上版本，在仓库根目录执行：

```bash
python -m venv .venv
```

Windows PowerShell激活环境：

```powershell
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

其他系统激活环境：

```bash
source .venv/bin/activate
python -m pip install -r requirements.txt
```

### 准备资料，不调用回答API

```bash
python public_manual/manual_project.py --prepare-only
```

首次从官方来源下载PDF，校验版本后提取26个块，生成的资料保存在 `public_manual/source_docs/`。此命令不加载BGE，也不请求DeepSeek。

### 运行单个问答

```bash
python public_manual/manual_project.py --question "事件日志最多记录多少个事件？" --top-k 3
```

现有命令行入口使用语义检索。它通过隐藏输入读取DeepSeek密钥并请求一次回答，按平台计费；首次检索还会下载BGE模型到本地。混合检索在 [hybrid.ipynb](public_manual/hybrid.ipynb) 中运行，人工检索表达诊断在 [query_diagnosis.ipynb](public_manual/query_diagnosis.ipynb) 中运行。

在VS Code或Jupyter中打开 `public_manual/` 下的Notebook，选择安装依赖的内核。Notebook注明了请求次数；重新生成后，应按新结果重新填写人工评价。新运行结果写入 `public_manual/results/runs/`，保留已提交的历史评测文件。

### 不调用API查看已有结果

直接在GitHub打开上述Notebook即可看到保存的运行输出，无需安装、密钥或模型下载。也可以在本地只读取结果文件：

```bash
python -c "import json; from pathlib import Path; rows=json.loads(Path('public_manual/results/llm09_new5_20261013_121225_300020.json').read_text(encoding='utf-8')); [print(r['id'], r['answer'], r['review'], sep='\n') for r in rows]"
```

## 官方来源、版本与局限

- 官方文档：[Easy UPS On-Line SRV系列中文用户手册](https://www.se.com/tw/zh/download/document/SPD_AHUG-AS9C3R_CH/)，官方文件名 `EUO-UM-990-91063B_CH.pdf`，Rev B。程序中的本地文件名为 `EasyUPS_SRV_Chinese_990-91063B.pdf`。
- 固定版本SHA256：`07fe222c86fcdfbee67f581c9e0e69a969820c379b7198b3b6921f9017b1dc7c`。程序同时检查22页；官方文件变化时会停止准备流程，需要重新核对标准。
- 原文版权归厂商，PDF、生成的原文块、模型文件、密钥和本地环境均不上传。结果文件中的摘录用于复核实验。
- PDF表格的文本提取可能丢失列关系；字符切块可能截断依据，命中相关页不等于找到答案。仍需打开原PDF核对。
- 固定15题参与了调试；新五题只是一次小检查，使用后也进入了已知问题集。模型重新生成可能变化，没有自动评分或完整手册覆盖。
- 静态手册无法证明设备当前电量、密码、当天当地价格。后续方向是检查其他新问题，比较切分与检索表达的影响，并继续保存失败和人工核对记录。

本次独立整理的验证范围与迁移来源见 [VALIDATION.md](VALIDATION.md)。本次没有重新请求DeepSeek。
