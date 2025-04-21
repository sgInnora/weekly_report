# Qwen2.5-Coder-7B 大模型微调完全指南

> 基于RunPod环境与LLaMA-Factory框架的全参数微调实战教程

## 目录

- [1. 环境准备](#1-环境准备)
  - [1.1 RunPod环境配置](#11-runpod环境配置)
  - [1.2 LLaMA-Factory安装](#12-llama-factory安装)
  - [1.3 依赖项安装](#13-依赖项安装)
- [2. 模型与数据准备](#2-模型与数据准备)
  - [2.1 模型下载](#21-模型下载)
  - [2.2 数据集准备](#22-数据集准备)
  - [2.3 数据格式要求](#23-数据格式要求)
- [3. 训练配置详解](#3-训练配置详解)
  - [3.1 全参数微调vs LoRA](#31-全参数微调vs-lora)
  - [3.2 训练脚本配置](#32-训练脚本配置)
  - [3.3 DeepSpeed配置](#33-deepspeed配置)
  - [3.4 多GPU训练设置](#34-多gpu训练设置)
- [4. 训练过程管理](#4-训练过程管理)
  - [4.1 磁盘空间管理](#41-磁盘空间管理)
  - [4.2 检查点管理](#42-检查点管理)
  - [4.3 训练过程监控](#43-训练过程监控)
  - [4.4 使用tmux进行后台训练](#44-使用tmux进行后台训练)
- [5. 常见问题与解决方案](#5-常见问题与解决方案)
  - [5.1 显存不足](#51-显存不足)
  - [5.2 训练中断恢复](#52-训练中断恢复)
  - [5.3 数据处理问题](#53-数据处理问题)
  - [5.4 模型保存与加载](#54-模型保存与加载)
- [6. 进阶功能](#6-进阶功能)
  - [6.1 Telegram训练监控](#61-telegram训练监控)
  - [6.2 自动清理检查点](#62-自动清理检查点)
  - [6.3 训练后评估](#63-训练后评估)
- [7. 附录](#7-附录)
  - [7.1 完整训练脚本](#71-完整训练脚本)
  - [7.2 监控脚本](#72-监控脚本)
  - [7.3 资源链接](#73-资源链接)

## 1. 环境准备

### 1.1 RunPod环境配置

RunPod是一个提供GPU计算资源的云平台，非常适合大型模型的训练。对于Qwen2.5-Coder-7B全参数微调，推荐以下配置：

- **GPU**：至少3×NVIDIA H100 (80GB)或同等性能GPU
- **RAM**：至少128GB
- **存储**：至少1TB SSD（训练数据、模型权重和检查点需要大量空间）
- **镜像**：PyTorch 2.0以上 + CUDA 11.8以上

在RunPod上创建Pod时，选择包含PyTorch的镜像，并确保分配足够的存储空间。对于初学者，推荐选择官方的PyTorch+CUDA模板。

登录RunPod实例后，首先更新系统并安装必要工具：

```bash
apt-get update
apt-get install -y git wget curl tmux htop
```

### 1.2 LLaMA-Factory安装

LLaMA-Factory是一个功能齐全的开源框架，专为大语言模型微调设计。克隆仓库并设置：

```bash
cd /workspace
git clone https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory
```

### 1.3 依赖项安装

创建虚拟环境并安装依赖：

```bash
python -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -e .
```

安装额外依赖：

```bash
# 安装DeepSpeed
pip install deepspeed==0.16.7

# 安装训练监控和可视化工具
pip install requests
```

## 2. 模型与数据准备

### 2.1 模型下载

从Hugging Face下载Qwen2.5-Coder-7B模型：

```bash
mkdir -p /workspace/models
cd /workspace/models

# 使用Hugging Face CLI下载
pip install -U huggingface_hub
huggingface-cli download Qwen/Qwen2.5-Coder-7B --local-dir Qwen2.5-Coder-7B --local-dir-use-symlinks False
```

> **注意**：如果遇到网络问题，可以使用`HTTPS_PROXY`环境变量或考虑先下载到本地再上传到RunPod。

验证模型文件是否下载完整：

```bash
ls -la /workspace/models/Qwen2.5-Coder-7B
```

确保包含以下关键文件：
- `config.json`
- `model.safetensors.index.json`
- `tokenizer.json`
- 以及其他模型分片文件（如`model-00001-of-00004.safetensors`）

### 2.2 数据集准备

创建数据目录并准备您的数据集：

```bash
mkdir -p /workspace/data/finetune_qwen_clean
```

可以通过多种方式将数据上传到RunPod：
- 使用`scp`命令从本地上传
- 从公共URL下载
- 通过RunPod文件管理器上传

### 2.3 数据格式要求

LLaMA-Factory支持多种数据格式，但对于指令微调，推荐使用Alpaca格式：

创建一个`dataset_info.json`文件，指定数据集结构：

```json
{
  "custom": {
    "file_name": "train_clean.json",
    "file_sha1": null,
    "columns": {
      "prompt": "instruction",
      "query": "input",
      "response": "output"
    },
    "type": "alpaca"
  }
}
```

然后准备训练数据`train_clean.json`，每行一个JSON对象：

```json
{"instruction":"请根据以下提示完成任务","input":"具体输入内容...","output":"期望的输出内容..."}
```

数据字段说明：
- `instruction`：任务指令，告诉模型要做什么
- `input`：具体的输入内容（可选，如果任务不需要额外输入可为空）
- `output`：期望模型生成的输出

数据集大小建议：大于50万条样本以获得良好的微调效果。

## 3. 训练配置详解

### 3.1 全参数微调vs LoRA

LLaMA-Factory支持多种微调方法：

| 方法 | 优点 | 缺点 | 推荐场景 |
|------|------|------|----------|
| 全参数微调 | 性能更好，模型能力更全面 | 需要大量GPU资源 | 有足够计算资源，追求最佳效果 |
| LoRA | 资源需求小，训练速度快 | 效果可能略差 | 资源有限，快速实验 |
| QLoRA | 节省显存，适用于消费级GPU | 训练速度较慢 | 个人设备，资源严重受限 |

本教程选择**全参数微调**，因为：
1. 我们有足够的GPU资源（3×H100）
2. 追求最佳微调效果
3. Qwen2.5-Coder-7B规模适中，全参数微调可行

### 3.2 训练脚本配置

创建自动化训练脚本`auto_gpu_train.sh`：

```bash
#!/bin/bash
set -e  # 出错时停止脚本执行

# 激活虚拟环境
echo "激活Python虚拟环境..."
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/venv/bin/activate"

# 检查环境是否正确激活
if ! command -v accelerate &> /dev/null; then
  echo "错误: 虚拟环境未正确激活，找不到accelerate命令"
  echo "尝试安装必要的依赖..."
  pip install -r requirements.txt
  if ! command -v accelerate &> /dev/null; then
    echo "仍然无法找到accelerate命令，请手动检查虚拟环境"
    exit 1
  fi
fi

# 检查磁盘空间
free_space=$(df -BG /workspace | awk 'NR==2 {print $4}' | sed 's/G//')
if [ "$free_space" -lt 100 ]; then
  echo "警告: 可用空间小于100GB，当前可用: ${free_space}GB"
  echo "清理旧的检查点..."
  python3 disk_monitor.py --clean-only
fi

# 设置GPU和训练参数
TOTAL_GPUS=$(nvidia-smi -L | wc -l)
BATCH_SIZE=1
GRAD_ACC_STEPS=16

# 使用checkpoint-500作为恢复点
CHECKPOINT_PATH="/workspace/outputs/qwen2.5-full/checkpoint-500"
RESUME_OPTION=""

if [ -d "$CHECKPOINT_PATH" ]; then
  # 检查checkpoint-500目录是否完整
  if [ -f "$CHECKPOINT_PATH/trainer_state.json" ]; then
    echo "发现检查点500，将从 $CHECKPOINT_PATH 恢复训练"
    RESUME_OPTION="--resume_from_checkpoint $CHECKPOINT_PATH"
  else
    echo "检查点目录不完整，将从头开始训练"
    # 可选：备份不完整的检查点
    mv "$CHECKPOINT_PATH" "${CHECKPOINT_PATH}_incomplete_$(date +%Y%m%d%H%M%S)"
    mkdir -p /workspace/outputs/qwen2.5-full
  fi
else
  echo "未发现检查点，将从头开始训练"
  mkdir -p /workspace/outputs/qwen2.5-full
fi

# 启动磁盘监控（在后台运行）
nohup python3 disk_monitor.py > disk_monitor.log 2>&1 &
MONITOR_PID=$!
echo "已启动磁盘监控，进程ID: $MONITOR_PID"

# 启动训练
echo "开始训练..."
accelerate launch \
  --num_processes=$TOTAL_GPUS \
  --mixed_precision=bf16 \
  --main_process_port=29501 \
  src/train.py \
  --stage sft \
  --model_name_or_path /workspace/models/Qwen2.5-Coder-7B \
  --dataset custom \
  --dataset_dir /workspace/data/finetune_qwen_clean \
  --template qwen \
  --finetuning_type full \
  --output_dir /workspace/outputs/qwen2.5-full \
  --cutoff_len 1024 \
  --preprocessing_num_workers 4 \
  --preprocessing_batch_size 256 \
  --overwrite_cache \
  --gradient_accumulation_steps $GRAD_ACC_STEPS \
  --per_device_train_batch_size $BATCH_SIZE \
  --learning_rate 1e-5 \
  --num_train_epochs 3 \
  --logging_steps 10 \
  --logging_first_step true \
  --save_steps 500 \
  --save_total_limit 3 \
  --bf16 \
  --gradient_checkpointing true \
  --deepspeed ds_config.json \
  --do_train \
  $RESUME_OPTION

# 训练结束后，停止磁盘监控
if [ -n "$MONITOR_PID" ]; then
  echo "训练完成，停止磁盘监控进程"
  kill $MONITOR_PID
fi

echo "训练已完成！"
# 退出虚拟环境
deactivate
```

### 3.3 DeepSpeed配置

创建DeepSpeed配置文件`ds_config.json`：

```json
{
  "zero_optimization": {
    "stage": 2,
    "offload_optimizer": {
      "device": "cpu",
      "pin_memory": true
    },
    "contiguous_gradients": true,
    "overlap_comm": true,
    "reduce_scatter": true,
    "allgather_partitions": true
  },
  "gradient_accumulation_steps": 16,
  "gradient_clipping": 1.0,
  "train_batch_size": 48,
  "train_micro_batch_size_per_gpu": 1,
  "steps_per_print": "inf",
  "bf16": {
    "enabled": true
  },
  "fp16": {
    "enabled": false
  },
  "zero_allow_untested_optimizer": true
}
```

主要配置说明：
- `zero_optimization.stage`：使用ZeRO-2优化，在多GPU训练时节省显存
- `offload_optimizer`：将优化器状态卸载到CPU，进一步节省GPU显存
- `gradient_accumulation_steps`：梯度累积步数，用于增加有效批量大小
- `train_batch_size`：总批量大小（所有GPU）
- `train_micro_batch_size_per_gpu`：每个GPU处理的样本数
- `bf16`：启用bfloat16混合精度训练，H100 GPU上性能最佳

### 3.4 多GPU训练设置

多GPU训练配置建议：

1. **设置NCCL参数**：对于多GPU通信，添加以下环境变量可提高性能：

```bash
export NCCL_DEBUG=INFO
export NCCL_IB_DISABLE=0
export NCCL_IB_GID_INDEX=3
export NCCL_NET_GDR_LEVEL=2
```

2. **GPU拓扑感知**：检查GPU拓扑并根据实际情况调整：

```bash
nvidia-smi topo -m
```

3. **实验和监控**：在训练初始阶段密切监控GPU利用率和内存使用，根据实际情况调整参数：

```bash
watch -n 1 nvidia-smi
```

## 4. 训练过程管理

### 4.1 磁盘空间管理

创建磁盘监控脚本`disk_monitor.py`：

```python
#!/usr/bin/env python3
import os
import time
import shutil
import argparse
import subprocess
from datetime import datetime

def get_disk_usage():
    """获取/workspace目录的磁盘使用情况"""
    result = subprocess.run(['df', '-h', '/workspace'], capture_output=True, text=True)
    return result.stdout.strip()

def get_folder_size(path):
    """获取文件夹大小"""
    total_size = 0
    for dirpath, dirnames, filenames in os.walk(path):
        for f in filenames:
            fp = os.path.join(dirpath, f)
            if os.path.exists(fp):
                total_size += os.path.getsize(fp)
    return total_size

def get_free_space_gb():
    """获取可用空间（GB）"""
    result = subprocess.run(['df', '-BG', '/workspace'], capture_output=True, text=True)
    lines = result.stdout.strip().split('\n')
    if len(lines) >= 2:
        parts = lines[1].split()
        if len(parts) >= 4:
            free_space = parts[3].replace('G', '')
            try:
                return int(free_space)
            except ValueError:
                return 0
    return 0

def clean_old_checkpoints(output_dir, keep_checkpoints=3):
    """清理旧的检查点，保留最新的N个"""
    if not os.path.exists(output_dir):
        print(f"输出目录 {output_dir} 不存在")
        return

    # 查找所有检查点目录
    checkpoint_dirs = []
    for item in os.listdir(output_dir):
        if item.startswith('checkpoint-') and os.path.isdir(os.path.join(output_dir, item)):
            try:
                checkpoint_num = int(item.split('-')[1])
                checkpoint_dirs.append((checkpoint_num, item))
            except ValueError:
                continue

    # 按检查点编号排序
    checkpoint_dirs.sort(reverse=True)

    # 保留最新的keep_checkpoints个，删除其余的
    if len(checkpoint_dirs) > keep_checkpoints:
        for _, dir_name in checkpoint_dirs[keep_checkpoints:]:
            dir_path = os.path.join(output_dir, dir_name)
            print(f"删除旧检查点: {dir_path}")
            try:
                shutil.rmtree(dir_path)
                print(f"成功删除检查点目录: {dir_path}")
            except Exception as e:
                print(f"删除失败: {e}")

def monitor_disk_space(threshold_gb=100, output_dir='/workspace/outputs/qwen2.5-full'):
    """监控磁盘空间，当低于阈值时清理旧检查点"""
    while True:
        free_space = get_free_space_gb()
        now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        print(f"[{now}] 当前可用磁盘空间: {free_space}GB")

        if free_space < threshold_gb:
            print(f"警告: 可用空间低于阈值({threshold_gb}GB)，清理旧检查点...")
            clean_old_checkpoints(output_dir)

            # 清理后再次检查
            new_free_space = get_free_space_gb()
            print(f"清理后可用空间: {new_free_space}GB")

            if new_free_space < threshold_gb:
                print("警告: 清理后空间仍然不足!")

        # 每小时检查一次
        time.sleep(3600)

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="磁盘空间监控工具")
    parser.add_argument("--clean-only", action="store_true", help="只清理旧检查点，不进行监控")
    parser.add_argument("--threshold", type=int, default=100, help="磁盘空间阈值（GB）")
    parser.add_argument("--output-dir", type=str, default="/workspace/outputs/qwen2.5-full", help="输出目录")
    args = parser.parse_args()

    if args.clean_only:
        print("清理旧检查点...")
        clean_old_checkpoints(args.output_dir)
        print(get_disk_usage())
    else:
        print(f"启动磁盘空间监控，阈值: {args.threshold}GB")
        print(get_disk_usage())
        monitor_disk_space(args.threshold, args.output_dir)
```

在训练前设置脚本权限：

```bash
chmod +x disk_monitor.py
```

### 4.2 检查点管理

LLaMA-Factory默认会创建检查点，但管理策略需要特别注意：

1. **检查点频率**：
   建议每500-1000步保存一次检查点，平衡恢复能力和磁盘使用。

2. **检查点限制**：
   通过`--save_total_limit 3`限制保存的检查点数量，防止占用过多空间。

3. **检查点断点续训**：
   若训练中断，可通过`--resume_from_checkpoint`参数指定恢复点：

```bash
CHECKPOINT_PATH="/workspace/outputs/qwen2.5-full/checkpoint-500"
# 确保检查点完整
if [ -f "$CHECKPOINT_PATH/trainer_state.json" ]; then
  RESUME_OPTION="--resume_from_checkpoint $CHECKPOINT_PATH"
fi
```

### 4.3 训练过程监控

创建训练监控日志查看工具：

```bash
# 查看最新损失和学习率
tail -f /workspace/outputs/qwen2.5-full/trainer_log.jsonl

# 查看GPU使用情况
watch -n 2 nvidia-smi
```

### 4.4 使用tmux进行后台训练

使用tmux确保训练在SSH断开后继续运行：

```bash
# 安装tmux（如需）
apt-get update && apt-get install -y tmux

# 创建新会话
tmux new-session -d -s training "cd /workspace/LLaMA-Factory && ./auto_gpu_train.sh"

# 列出会话
tmux list-sessions

# 连接到会话
tmux attach-session -t training

# 分离会话（在tmux内使用Ctrl+B后按D）
```

## 5. 常见问题与解决方案

### 5.1 显存不足

如果遇到显存不足（CUDA Out of Memory）:

1. **减小模型上下文长度**：
   ```bash
   --cutoff_len 512  # 从1024减小到512
   ```

2. **启用梯度检查点**：
   ```bash
   --gradient_checkpointing true
   ```

3. **启用优化器卸载**：
   在DeepSpeed配置中启用CPU卸载：
   ```json
   "offload_optimizer": {
     "device": "cpu",
     "pin_memory": true
   }
   ```

4. **减小批量大小**：
   降低`per_device_train_batch_size`和增加`gradient_accumulation_steps`

### 5.2 训练中断恢复

如果训练中断：

1. **检查日志文件**：
   ```bash
   tail -n 100 /workspace/LLaMA-Factory/logs/*.log
   ```

2. **验证检查点完整性**：
   ```bash
   ls -la /workspace/outputs/qwen2.5-full/checkpoint-XXX/
   ```
   确保包含`trainer_state.json`文件

3. **修改恢复点路径**：
   在`auto_gpu_train.sh`中更新`CHECKPOINT_PATH`变量

4. **清除锁文件**（必要时）：
   ```bash
   find /workspace/outputs -name "*.lock" -delete
   ```

### 5.3 数据处理问题

解决常见数据问题：

1. **数据格式错误**：
   确保JSON格式正确，无额外逗号，所有键名有引号

2. **字段名不匹配**：
   检查`dataset_info.json`中的字段映射是否与实际数据匹配

3. **OOM预处理**：
   如果数据预处理OOM，减小`preprocessing_batch_size`

4. **验证数据集**：
   ```bash
   python -c "import json; data=json.load(open('/workspace/data/finetune_qwen_clean/train_clean.json')); print(f'样本数: {len(data)}')"
   ```

### 5.4 模型保存与加载

模型保存与转换：

1. **检查生成的模型**：
   ```bash
   ls -la /workspace/outputs/qwen2.5-full/
   ```

2. **将模型转为HF格式**（非DeepSpeed格式）：
   ```bash
   python -c "from transformers import AutoModelForCausalLM; model = AutoModelForCausalLM.from_pretrained('/workspace/outputs/qwen2.5-full/'); model.save_pretrained('/workspace/outputs/qwen2.5-full-hf')"
   ```

3. **测试模型生成**：
   ```bash
   python src/cli_demo.py \
     --model_name_or_path /workspace/outputs/qwen2.5-full \
     --template qwen
   ```

## 6. 进阶功能

### 6.1 Telegram训练监控

创建Telegram监控脚本`training_monitor.py`：

```python
#!/usr/bin/env python3
import os
import time
import json
import datetime
import subprocess
import requests
import threading

# Telegram配置
TELEGRAM_BOT_TOKEN = "YOUR_BOT_TOKEN"
TELEGRAM_CHAT_ID = "YOUR_CHAT_ID"
MONITOR_INTERVAL = 20 * 60  # 20分钟的间隔，以秒为单位

def send_telegram_message(message):
    """发送消息到Telegram"""
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    data = {
        "chat_id": TELEGRAM_CHAT_ID,
        "text": message,
        "parse_mode": "Markdown"
    }

    try:
        response = requests.post(url, data=data)
        if response.status_code != 200:
            print(f"发送Telegram消息失败: {response.text}")
        return response.json()
    except Exception as e:
        print(f"发送Telegram消息出错: {str(e)}")
        return None

def get_gpu_status():
    """获取GPU状态信息"""
    try:
        result = subprocess.run(['nvidia-smi', '--query-gpu=index,utilization.gpu,memory.used,memory.total,temperature.gpu',
                                '--format=csv,noheader,nounits'],
                                capture_output=True, text=True, check=True)

        gpu_stats = []
        for line in result.stdout.strip().split('\n'):
            if line:
                parts = line.split(', ')
                if len(parts) >= 5:
                    gpu_idx, gpu_util, mem_used, mem_total, temp = parts
                    gpu_stats.append({
                        "index": gpu_idx,
                        "utilization": gpu_util + "%",
                        "memory": f"{mem_used}/{mem_total} MiB",
                        "temperature": temp + "°C"
                    })

        return gpu_stats
    except Exception as e:
        return f"获取GPU状态失败: {str(e)}"

def get_training_processes():
    """获取训练进程状态"""
    try:
        result = subprocess.run(['ps', 'aux'], capture_output=True, text=True, check=True)
        processes = result.stdout.strip().split('\n')

        training_processes = []
        for process in processes:
            if 'python' in process and ('accelerate' in process or 'train.py' in process) and 'grep' not in process:
                training_processes.append(process)

        return training_processes
    except Exception as e:
        return f"获取训练进程失败: {str(e)}"

def get_tmux_training_status():
    """获取tmux会话中的训练日志状态"""
    try:
        result = subprocess.run(['tmux', 'capture-pane', '-p', '-t', 'training'],
                             capture_output=True, text=True, check=True)

        log_lines = result.stdout.strip().split('\n')
        recent_logs = log_lines[-20:]  # 获取最后20行日志

        interesting_logs = []
        for line in recent_logs:
            if any(keyword in line for keyword in ['loss', 'step', 'epoch', 'error', 'Exception', 'ValueError', 'GPU']):
                interesting_logs.append(line)

        return interesting_logs if interesting_logs else ["未找到相关训练日志"]
    except Exception as e:
        return [f"获取tmux训练状态失败: {str(e)}"]

def check_disk_space():
    """检查磁盘空间"""
    try:
        result = subprocess.run(['df', '-h', '/workspace'], capture_output=True, text=True, check=True)
        return result.stdout.strip()
    except Exception as e:
        return f"检查磁盘空间失败: {str(e)}"

def check_training_status():
    """检查训练状态并发送通知"""
    current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    # 构建状态报告
    report = [f"🤖 *训练状态报告* - {current_time}"]

    # 获取GPU状态
    gpu_stats = get_gpu_status()
    if isinstance(gpu_stats, list):
        report.append("\n📊 *GPU状态*:")
        for gpu in gpu_stats:
            report.append(f"GPU {gpu['index']}: 利用率 {gpu['utilization']}, 内存 {gpu['memory']}, 温度 {gpu['temperature']}")
    else:
        report.append(f"\n⚠️ GPU状态: {gpu_stats}")

    # 获取训练进程
    training_processes = get_training_processes()
    if isinstance(training_processes, list):
        num_processes = len(training_processes)
        report.append(f"\n🔄 *训练进程*: 发现 {num_processes} 个训练相关进程")
    else:
        report.append(f"\n⚠️ 训练进程: {training_processes}")

    # 获取tmux训练状态
    tmux_logs = get_tmux_training_status()
    report.append("\n📝 *最近训练日志*:")
    for log in tmux_logs[-5:]:  # 仅显示最后5行相关日志
        report.append(log)

    # 检查磁盘空间
    disk_space = check_disk_space()
    report.append(f"\n💾 *磁盘空间*:\n{disk_space}")

    # 检查异常情况
    has_error = False
    if isinstance(gpu_stats, list):
        for gpu in gpu_stats:
            if int(gpu["utilization"].replace("%", "")) < 10:  # GPU利用率低于10%
                has_error = True
                report.append(f"\n⚠️ *警告*: GPU {gpu['index']} 利用率过低 ({gpu['utilization']})")

    if isinstance(training_processes, list) and len(training_processes) == 0:
        has_error = True
        report.append("\n🚨 *警告*: 未检测到训练进程！")

    for log in tmux_logs:
        if any(err in log for err in ["error", "Error", "ERROR", "Exception", "失败", "CUDA out of memory"]):
            has_error = True
            report.append(f"\n🚨 *错误*: 训练日志中发现错误: {log}")

    # 发送消息
    message = "\n".join(report)
    if has_error:
        message = "⚠️ *训练异常警报* ⚠️\n" + message

    send_telegram_message(message)

    # 输出到控制台
    print(f"状态报告已发送到Telegram ({current_time})")
    if has_error:
        print("检测到训练异常!")

    return has_error

def monitor_loop():
    """监控循环"""
    print(f"启动训练监控，间隔: {MONITOR_INTERVAL // 60} 分钟")

    # 发送初始化消息
    send_telegram_message("🔔 *训练监控已启动*\n\n将每20分钟发送一次状态更新，异常情况会立即通知。")

    # 立即执行第一次检查
    has_error = check_training_status()

    while True:
        try:
            # 等待指定的时间
            time.sleep(MONITOR_INTERVAL)
            has_error = check_training_status()

            # 如果检测到错误，则提高监控频率
            if has_error:
                time.sleep(120)  # 2分钟后再次检查
                check_training_status()

        except KeyboardInterrupt:
            print("监控终止")
            send_telegram_message("🛑 *训练监控已停止*")
            break
        except Exception as e:
            error_msg = f"监控脚本出错: {str(e)}"
            print(error_msg)
            send_telegram_message(f"⚠️ *监控脚本错误*\n\n{error_msg}")
            time.sleep(300)  # 出错后等待5分钟再继续

if __name__ == "__main__":
    # 启动监控线程
    monitor_thread = threading.Thread(target=monitor_loop, daemon=True)
    monitor_thread.start()

    try:
        # 保持主线程运行
        while True:
            time.sleep(60)
    except KeyboardInterrupt:
        print("监控程序退出")
    finally:
        send_telegram_message("🛑 *训练监控已停止*")
```

在tmux会话中运行监控脚本：

```bash
tmux new-session -d -s monitor "/workspace/training_monitor.py"
```

### 6.2 自动清理检查点

自动清理旧检查点的策略：

1. 在`auto_gpu_train.sh`中已集成磁盘空间监控和清理
2. 训练参数`--save_total_limit 3`限制保存检查点数量
3. 设置`disk_monitor.py`定期检查磁盘空间，阈值低于100GB时清理旧检查点

### 6.3 训练后评估

训练完成后评估模型：

```bash
# 在单个示例上测试
python src/cli_demo.py \
  --model_name_or_path /workspace/outputs/qwen2.5-full \
  --template qwen

# 进行基准评估
python src/evaluate.py \
  --model_name_or_path /workspace/outputs/qwen2.5-full \
  --template qwen \
  --task mmlu \
  --split test \
  --n_shot 5 \
  --batch_size 4
```

## 7. 附录

### 7.1 完整训练脚本

完整的`auto_gpu_train.sh`脚本已在上文提供。

### 7.2 监控脚本

完整的`training_monitor.py`和`disk_monitor.py`脚本已在上文提供。

### 7.3 资源链接

- [LLaMA-Factory官方文档](https://github.com/hiyouga/LLaMA-Factory)
- [Qwen2.5-Coder-7B模型页面](https://huggingface.co/Qwen/Qwen2.5-Coder-7B)
- [DeepSpeed文档](https://www.deepspeed.ai/docs/config-json/)
- [RunPod文档](https://docs.runpod.io/)

---

本指南希望能帮助您成功完成Qwen2.5-Coder-7B的微调。如有问题或需要进一步的帮助，请查阅相关文档或寻求社区支持。祝您训练顺利！
