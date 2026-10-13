# 工业产品手册 RAG 问答原型

基于施耐德 Easy UPS 公开中文手册的个人复现实验。目标是从手册中检索依据、回答问题并保留可核对的页码引用，重点观察漏检、错误前提和资料缺失。项目与厂商没有官方合作关系。

## 目标与当前范围

当前使用一份22页手册中的7页：PDF第4、5、6、11、14、15、16页，生成26个文档块。内容包括规格、软件通信、显示设置和事件日志。

程序、问题集、运行结果和带输出的Notebook放在 `public_manual/`。

## 问答流程

**PDF解析 → 保留页码的切分 → 语义或混合检索 → DeepSeek回答 → 核对原文与引用**

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

两路检索都使用问题文本，预期答案和页码只用于事后核对。RRF分数用于融合排名。运行和对比见 [hybrid.ipynb](public_manual/hybrid.ipynb)。

## 评测结果与失败案例

以下结果来自各轮的人工核对。固定15题参与了调试，新五题是一次小样本检查，各轮结果分别列出，不作为系统整体准确率。

`status=ok`表示请求完成，`reference_page_hit`表示命中预期页；回答是否正确还需要核对具体原文。

| 轮次 | 问题范围 | 结果 | 记录 |
|---|---|---|---|
| 语义检索基线 | 15道固定题 | 当次人工核对摘要为12/15符合预期，R03、R04、R09漏答；JSON中另有R01、R05的逐题评价。 | [基线15题](public_manual/results/llm07_all15.json) |
| 混合检索改进 | 同一15道固定题，参与了调试 | R03、R04、R09三题修复通过；其余12题未逐条新增人工评价。 | [当轮对比基线](public_manual/results/llm08_baseline15.json)、[混合检索15题](public_manual/results/llm08_all15.json) |
| 新问题检查 | 5道未参与此前调试的新题 | 单次人工核对4/5通过，U03未检索到贮存充电条款。 | [五题和标准](public_manual/unseen_cases09.json)、[首次结果与五条评价](public_manual/results/llm09_new5_20261013_121225_300020.json) |
| 人工查询表达诊断 | U03已进入调试 | 人工调整U03的检索表达后找到依据，回答通过核对。 | [诊断程序](public_manual/query_diagnosis.py)、[诊断结果](public_manual/results/llm10_query_diagnosis_20261013_131350_959665.json) |

代表性失败及定位：

- **R03、R04**：候选命中了第15页，但没有包含默认设置表所在的块01。混合检索将该块加入候选，回答分别给出报警启用、绿色模式禁用。
- **R09**：候选命中了第16页块02，却没有含日志数量的块01。混合检索补进块01，回答给出最后10个事件。
- **U03**：首次候选为 `p06-c02`、`p15-c02`、`p06-c01`，没有第4页的贮存充电条款。人工检索表达“电池贮存期间的充电要求和充电周期”找到了 `p04-c02`，新回答给出每六个月充电一次。这次使用的是人工给定的检索表达。

细节与已有输出见 [新问题检查](public_manual/new_questions.ipynb) 和 [查询表达诊断](public_manual/query_diagnosis.ipynb)。U03的后续诊断单独保存，首次五题检查仍记录为4/5；目前未实现自动查询改写。

[取1块与3块的对比](public_manual/results/llm07_topk_comparison.json) 中，PowerChute问题两组回答均符合依据，参考文字由250字符增至872字符；增加候选数量后，这道题的回答没有改善。

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

在VS Code或Jupyter中打开 `public_manual/` 下的Notebook，选择安装依赖的内核。Notebook注明了请求次数。保存的评价对应当次运行，重跑后需要重新核对。新结果写入 `public_manual/results/runs/`。

四份Notebook保留了已有输出，路径中的 `<项目目录>` 是原本机路径的替代写法。历史评测保存在 `public_manual/results/`，新运行不会覆盖这些文件。

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
- 当前仅覆盖选定的7页，评价依靠人工核对。模型重新生成的回答可能变化，后续需要用其他新问题继续检查。
- 静态手册无法证明设备当前电量、密码、当天当地价格。后续方向是检查其他新问题，比较切分与检索表达的影响，并继续保存失败和人工核对记录。

运行检查、缓存使用情况和未验证项见 [验证记录](VALIDATION.md)。
