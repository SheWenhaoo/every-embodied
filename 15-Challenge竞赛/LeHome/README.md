# LeHome Challenge 比赛镜像使用教程

本文档面向已经拿到官方 LeHome 比赛镜像、希望尽快开始训练与评测的同学。

重点回答四个问题：

1. 当前镜像环境能不能直接用。
2. 训练应该下载 `merged` 还是非 `merged` 数据。
3. 如何用最小代价启动训练、评测、复现实验。
4. 训练配置具体怎么写，输出放到哪里。
5. 哪些大文件不要提交进教程仓库。

---

## 1. 当前镜像是否可用

结论：可用。

我已在当前镜像中完成以下验证：

- `LeHome` 仓库路径：`/root/lehome-challenge`
- `isaacsim` 可用：`5.1.0.0`
- `lerobot` 可用：`0.4.3`
- GPU：`NVIDIA L40`
- 本地 HEAD：`2953e2f9c1376a1e49ac10b3ad690efb886f4c6c`
- 当前远端 `origin/main`：`32b53595da504880592a79ed5e362ad0ba0fac6b`
- 已成功跑通一轮 ACT 评测
- 已成功跑通一轮 DP 评测，并且 `--policy_num_inference_steps 1` 生效
- 旧的失败评测视频已清理，后续统一输出到：
  - `/root/gpufree-data/lehome-outputs/eval/`

说明：

- 环境是能启动、能加载模型、能建场景、能完成评测的。
- 当前本地代码不是远端最新。
- 如果大家想严格对齐当前镜像，请先切到：

```bash
git checkout 2953e2f9c1376a1e49ac10b3ad690efb886f4c6c
```

- 如果大家想跟远端最新主分支对齐，再切到：

```bash
git fetch origin
git checkout 32b53595da504880592a79ed5e362ad0ba0fac6b
```

- 当前复现实验结果与官方复现包描述一致，成功率依旧偏低，这更像是任务本身/物理效果问题，不是镜像起不来。

---

## 2. 训练数据该下载哪个

### 2.1 结论

如果目标是尽快开始基线训练，优先下载：

```bash
hf download lehome/dataset_challenge_merged --repo-type dataset --local-dir Datasets/example
```

也就是优先使用 `merged` 版本。

### 2.2 原因

官方训练配置默认就是 `merged` 数据：

- `configs/train_act.yaml` 使用 `Datasets/example/top_long_merged`
- `configs/train_dp.yaml` 使用 `Datasets/example/top_long_merged`
- `configs/train_smolvla.yaml` 使用 `Datasets/example/top_long_merged`

这说明官方预期的训练入口就是“合并后的数据集”。

### 2.3 `merged` 和非 `merged` 的区别

可以这样理解：

- `dataset_challenge`
  - 更偏原始/分散的数据组织。
  - 适合你自己做数据处理、拆分、增广、补深度、补点云。
  - 如果你想研究数据生产流程，或者自己重构训练集，可以用它。
- `dataset_challenge_merged`
  - 更偏训练直接可用的整理结果。
  - 已经按类别合并成 `top_long_merged`、`pant_long_merged` 这类目录。
  - 最适合直接喂给 `lerobot-train`。

### 2.4 深度数据要不要下载

如果你只是先跑通 baseline：

- 先用 `merged`
- 先不用深度
- 先用官方 RGB + joint state 基线

原因：

- 官方文档明确支持 `state + RGB` 作为已验证组合
- 深度虽然可用，但会增加存储、I/O 和配置复杂度
- 官方数据处理文档里还专门给了“去掉 depth 减小存储”的流程

### 2.5 推荐选择

推荐顺序：

1. 入门/比赛 baseline：`dataset_challenge_merged`
2. 需要深度实验：在 `merged` 基础上再加 `observation.top_depth`
3. 需要自己做数据重构：再研究非 `merged` 的 `dataset_challenge`

---

## 3. 最小启动流程

## 3.1 克隆官方仓库

```bash
git clone https://github.com/lehome-official/lehome-challenge.git
cd lehome-challenge
```

如果镜像已经预装好了，直接进入现成目录即可：

```bash
cd /root/lehome-challenge
```

## 3.2 下载资产

```bash
hf download lehome/asset_challenge --repo-type dataset --local-dir Assets
```

## 3.3 下载训练样例数据

```bash
hf download lehome/dataset_challenge_merged --repo-type dataset --local-dir Datasets/example
```

如果只想快速验证一个类别，也可以只下载需要的子目录，例如：

```bash
hf download lehome/dataset_challenge_merged \
  --repo-type dataset \
  --local-dir Datasets/example \
  --include 'top_long_merged/**'
```

## 3.4 训练 ACT

建议使用教程专用配置，统一把输出保存到数据盘：

- 配置文件：
  - `/root/gpufree-data/every-embodied/15-Challenge竞赛/LeHome/resources/configs/train_act_every_embodied.yaml`

```bash
mkdir -p /root/gpufree-data/lehome-outputs/train/act_top_long

lerobot-train \
  --config_path /root/gpufree-data/every-embodied/15-Challenge竞赛/LeHome/resources/configs/train_act_every_embodied.yaml \
  2>&1 | tee /root/gpufree-data/lehome-outputs/train/act_top_long/train.log
```

这条命令做了两件事：

1. 用教程专用 YAML 启动训练。
2. 用 `tee` 把终端日志同时保存到 `train.log`，方便后面分析 loss 和画图。

对于初学者，建议第一次不要改命令行，而是先读懂 YAML 里每个字段的作用，再只改一两个参数做实验。

## 3.5 训练 DP

同样建议使用教程专用配置：

- 配置文件：
  - `/root/gpufree-data/every-embodied/15-Challenge竞赛/LeHome/resources/configs/train_dp_every_embodied.yaml`

```bash
mkdir -p /root/gpufree-data/lehome-outputs/train/dp_top_long

lerobot-train \
  --config_path /root/gpufree-data/every-embodied/15-Challenge竞赛/LeHome/resources/configs/train_dp_every_embodied.yaml \
  2>&1 | tee /root/gpufree-data/lehome-outputs/train/dp_top_long/train.log
```

DP 相比 ACT 通常更慢、更吃资源，所以教程配置里把：

- `batch_size` 调小到 `8`
- `steps` 拉长到 `90000`
- `log_freq` 调细到 `100`

这样更符合 Diffusion Policy 的常见训练习惯，也更方便观察收敛过程。

## 3.6 评测 ACT

```bash
python -m scripts.eval \
  --policy_type lerobot \
  --policy_path /root/gpufree-data/lehome-outputs/train/act_top_long/checkpoints/last/pretrained_model \
  --dataset_root Datasets/example/top_long_merged \
  --garment_type top_long \
  --num_episodes 2 \
  --enable_cameras \
  --save_video \
  --video_dir /root/gpufree-data/lehome-outputs/eval/act_top_long \
  --device cpu
```

## 3.7 评测 DP

CPU 评测时建议显式限制 diffusion 推理步数：

```bash
python -m scripts.eval \
  --policy_type lerobot \
  --policy_path /root/gpufree-data/lehome-outputs/train/dp_top_long/checkpoints/last/pretrained_model \
  --dataset_root Datasets/example/top_long_merged \
  --garment_type top_long \
  --num_episodes 2 \
  --enable_cameras \
  --save_video \
  --video_dir /root/gpufree-data/lehome-outputs/eval/dp_top_long \
  --device cpu \
  --policy_device cpu \
  --policy_num_inference_steps 1
```

## 3.8 训练配置详解

教程专用配置里，最值得解释的是这些字段：

- `dataset.root`
  - 训练数据目录。
  - baseline 默认使用 `Datasets/example/top_long_merged`。
  - 这样设计的原因是官方基线配置本身就是指向 `*_merged`，初学者不需要先理解多数据集合并逻辑，也能直接开练。
- `policy.type`
  - `act` 或 `diffusion`。
  - ACT 是更推荐的第一站，因为训练逻辑更直观，速度通常也更快。
  - Diffusion Policy 更适合在你已经跑通 ACT baseline 后再继续比较。
- `policy.device`
  - 训练建议 `cuda`。
  - 因为图像编码器和策略网络都比较重，CPU 训练几乎没有效率。
- `input_features`
  - baseline 推荐 `observation.state + top/left/right RGB`。
  - 这是一个工程上很稳妥的起点。
  - `state` 提供机械臂关节信息，RGB 提供布料形态和位姿信息。
  - 对学习者来说，这个组合也最容易理解：一部分是“机器人自己当前姿态”，一部分是“摄像头看到的场景”。
- `output_features`
  - baseline 推荐输出 `action`。
  - 也就是直接预测关节动作。
  - 这种 joint-space control 比 `ee_pose` 更适合作为教学 baseline，因为它少了一层 IK 误差。
- `output_dir`
  - 已改到数据盘，例如 `/root/gpufree-data/lehome-outputs/train/act_top_long`。
  - 这样做是为了避免把系统盘写满。
- `batch_size`
  - 和显存直接相关。
  - ACT 当前建议 `16`，DP 当前建议 `8`。
  - 对初学者可以简单理解为：一次喂给模型多少条训练样本。
  - 越大通常训练越快，但显存占用也越高。
- `steps`
  - 总训练步数。
  - ACT 可先跑 `30000`，DP 建议更长，教程配置里给到 `90000`。
  - 训练步数不是“越大越好”，而是“足够看到收敛趋势”。
  - 教学上先用一个可跑完、可观察的步数更重要。
- `save_freq`
  - checkpoint 保存间隔。
  - 教程里调高了保存频率，便于中间结果回看。
  - 这对教学尤其重要，因为你可以比较不同阶段 checkpoint 的效果，而不是只看最后一个模型。
- `log_freq`
  - 日志打印频率。
  - 教程里调高到更细，方便后续画图和排障。
  - 对学习者来说，训练日志就是最直观的“训练过程观察窗口”。
- `eval_freq`
  - 训练中的阶段性评估间隔。
  - 便于观察并不是只有 loss 在变化。
  - 如果一个模型 loss 在下降，但 eval success rate 不涨，这通常意味着“学到了训练分布内拟合”，但未必真正学会了任务。

## 3.8.1 为什么 ACT 和 DP 的参数不一样

这是很多学习者第一次会问的问题。

### ACT

ACT 可以先理解成“基于视觉和状态做动作序列预测”的 transformer 类策略。

在本教程里，我们给 ACT 这样一组较保守的参数：

- `batch_size: 16`
- `steps: 30000`
- `save_freq: 5000`
- `log_freq: 200`
- `eval_freq: 5000`

这样设计的目的：

- 比较容易在单卡上跑起来
- 日志足够细，便于教学观察
- 每隔 `5000` step 就能拿到一个 checkpoint
- 总步数 `30000` 足够看到一个 baseline 的基本收敛趋势

### Diffusion Policy

Diffusion Policy 的推理和训练都更重，尤其在多相机输入下更明显。

教程配置里我们给了：

- `batch_size: 8`
- `steps: 90000`
- `save_freq: 10000`
- `log_freq: 100`
- `eval_freq: 10000`

这样设计的目的：

- 降低显存压力
- 延长训练步数，给 diffusion 模型足够的收敛空间
- 用更细的日志频率观察 loss 和 gradient norm

所以不要机械地认为“ACT 和 DP 的参数应该一样”。不同策略结构，本来就应该有不同的训练节奏。

## 3.8.2 训练要跑多久，显存大概多少

这里给的是经验预估，不是绝对值。

在当前这类 `top_long_merged + state + 3路RGB` 配置下，可以先这样估算：

### ACT

- 推荐显存：至少 `16GB`，更稳妥是 `24GB+`
- 在 L40 这类卡上通常可以单卡训练
- `30000 step` 一般是“小时级”任务，不是分钟级
- 如果日志里 `data_s` 和 `updt_s` 都不大，通常几个小时内能看到完整 baseline 结果

### DP

- 推荐显存：`24GB+` 更稳
- `90000 step` 通常明显比 ACT 更久
- 更适合作为过夜训练任务

学习者应该关心的是：

- 有没有 OOM
- 每 step 花多久
- loss 是否稳定下降
- 中间 checkpoint 的 eval 是否真的变好

不要一上来纠结“到底 3 小时还是 5 小时”，先看自己机器日志里的：

- `updt_s`
- `data_s`
- `step_per_sec`

这些值才是真正决定训练时间的依据。

## 3.8.3 第一次训练建议怎么做

推荐按下面顺序学：

1. 先训练 ACT，不要先上 DP。
2. 不要先加 depth。
3. 不要先改成 `ee_pose`。
4. 先跑通一个类别，比如 `top_long_merged`。
5. 先观察 loss、grad norm、lr 和中间 eval。
6. 确认 pipeline 稳定以后，再扩大到更多类别。

## 3.9 建议保留的训练中间结果

建议至少保留这些：

- `train.log`
- 所有 checkpoint
- `config.yaml` 或训练配置副本
- 训练曲线图
- 训练指标 CSV / JSON 摘要
- 若有阶段性 eval，也保留对应日志和视频

对于教学用途，还建议加上：

- 一张训练曲线截图
- 一份“最佳 checkpoint”和“最后 checkpoint”的对比结论
- 一份训练耗时记录
- 一份显存占用记录

建议统一放到：

```text
/root/gpufree-data/lehome-outputs/
├── train/
├── eval/
└── plots/
```

## 3.10 训练日志解析与绘图

教程已附带一个脚本：

- `/root/gpufree-data/every-embodied/15-Challenge竞赛/LeHome/resources/scripts/plot_train_metrics.py`

它会从 `train.log` 中解析并生成：

- `train_metrics.csv`
- `train_metrics_summary.json`
- `train_metrics.png`

示例：

```bash
python /root/gpufree-data/every-embodied/15-Challenge竞赛/LeHome/resources/scripts/plot_train_metrics.py \
  --log_file /root/gpufree-data/lehome-outputs/train/act_top_long/train.log \
  --out_dir /root/gpufree-data/lehome-outputs/plots/act_top_long \
  --title "ACT Top-Long Training Metrics"
```

当前脚本默认记录这些指标：

- `loss`
- `grdn`，也就是 gradient norm
- `lr`
- `updt_s`
- `data_s`
- `step_per_sec`

这些指标的教学意义分别是：

- `loss`
  - 最基础的优化目标，先看是否整体下降。
- `grdn`
  - 用来看梯度是否爆炸或异常不稳定。
- `lr`
  - 用来看学习率调度是否正常。
- `updt_s`
  - 单步参数更新耗时。
- `data_s`
  - 单步数据加载耗时。
- `step_per_sec`
  - 训练吞吐量的直观指标。

## 3.11 除了 loss，还建议看哪些指标

如果站在 ACT / Diffusion Policy 论文复现和工程训练角度，除了 loss，还建议尽量记录：

- 训练 loss
- 验证 loss
- gradient norm
- learning rate
- data loading time
- update time
- steps/sec 或 samples/sec
- checkpoint 对应的阶段性 eval 成功率
- 平均回报 `return`
- episode length
- 不同 garment/category 的 success rate

真正和比赛结果最相关的，不是单独看 loss，而是：

1. `loss` 是否稳定下降
2. `gradient norm` 是否异常爆炸
3. `lr` 调度是否正常
4. 中间 checkpoint 的 `eval success rate` 是否同步提升

## 3.12 训练结束后会保存什么

训练完成后，通常最值得关注的是这些内容：

- `train.log`
  - 原始训练日志
- `checkpoints/`
  - 每个阶段保存下来的模型
- `last/`
  - 最后一次训练状态
- `pretrained_model/`
  - 用于评测和部署的模型目录
- 你自己生成的：
  - `train_metrics.csv`
  - `train_metrics_summary.json`
  - `train_metrics.png`

从教学角度，建议把这些文件分成两类理解：

### 第一类：训练产物

- checkpoint
- last
- pretrained_model

这些是“模型本身”。

### 第二类：训练证据

- train.log
- 指标 CSV/JSON
- 曲线图
- 中间 eval 结果

这些是“为什么我们认为这个模型训练正常”的证据。

教学里两类都要保留，不能只保留模型文件。

如果后续你要再往上加，我建议优先补：

- 每个 checkpoint 自动 eval
- 每个类别的 success rate 曲线
- best checkpoint 自动挑选

---

## 4. 推荐训练策略

### 4.1 baseline 配置

建议先从下面这套开始：

- policy: `ACT`
- 输入：`observation.state + top/left/right RGB`
- 输出：`action`
- device: 训练时用 `cuda`
- 评测时 env 用 `cpu`

### 4.2 为什么不建议一上来就上 depth

- 深度会增大数据量和训练复杂度
- baseline 先跑通更重要
- 官方文档明确说 `state + RGB` 已验证可用
- 如果要加 depth，建议在 baseline 收敛后做对照实验

### 4.3 是否建议用 EE pose

不建议作为首选。

官方文档也明确提醒：

- `observation.ee_pose`
- `action.ee_pose`

由于 IK 和硬件限制，稳定性不如 joint-space control。比赛 baseline 更建议用：

- `observation.state`
- `action`

---

## 5. 清理与仓库控制

如果你要把这个教程整理进自己的大仓库，只建议保留这些轻量内容：

- 教程文档
- 训练配置样例
- 评测命令
- 少量日志片段
- 少量截图或短视频链接
- 必要的 patch / overlay 说明

不建议提交这些大资源：

- `Assets/`
- `Datasets/`
- `outputs/`
- `logs/`
- `models/`
- `videos/`
- `plots/`
- `.cache/`
- 任何大于几十 MB 的权重和素材

原因很简单：这些内容都应该通过下载命令获取，不应该进教程子仓库。

### 5.1 之前失败的评测结果

如果只是教程清理，之前那批 `0%` 成功率的失败视频可以删除。

当前需要删除的目录是：

```text
/root/gpufree-data/lehome-eval-outputs/
```

后续统一改用：

```text
/root/gpufree-data/lehome-outputs/eval/
```

---

## 6. 建议的仓库组织方式

建议在 `every-embodied` 中新增如下结构：

```text
15-Challenge竞赛/
└── LeHome/
    ├── README.md
    ├── .gitignore
    └── resources/
        ├── commands.md
        ├── configs/
        └── scripts/
```

如果后续你还会增加其他比赛，也可以沿用同样的结构。

---

## 7. 可直接复用的检查命令

## 7.1 检查 GPU

```bash
nvidia-smi
```

## 7.2 检查核心包

```bash
python - <<'PY'
import isaacsim, lerobot
print("isaacsim ok")
print("lerobot", getattr(lerobot, "__version__", "unknown"))
PY
```

## 7.3 检查官方资产是否已下载

```bash
ls Assets
ls Datasets/example
```

## 7.4 检查单件服装评测列表

```bash
cat Assets/objects/Challenge_Garment/Release/Release_test_list.txt
```

---

## 8. 镜像发布建议

如果你要宣传这个镜像，建议在说明里强调：

- 已预装官方 LeHome 环境
- 已验证可跑通训练与评测
- 建议首次使用时下载 `asset_challenge` + `dataset_challenge_merged`
- 已适配 `L40` / 服务器场景
- 提供 ACT/DP baseline 启动命令
- 默认不把比赛大资源打进教程仓库

可直接对外描述为：

> 一个面向 LeHome Challenge 的开箱即用镜像，预装 Isaac Sim / LeRobot / LeHome 运行环境，提供训练、评测、数据下载和问题复现的最小闭环。

---

## 9. 本地验证结果

本地已经验证：

- ACT 单 garment 评测可跑通，结果 `Success Rate = 0.00%`
- DP 单 garment 评测可跑通，且 `policy_num_inference_steps=1` 能生效
- 评测视频可以成功导出

这说明：

- 镜像可用
- 训练/评测链路可用
- 教程可以基于这套镜像直接编写和发布
