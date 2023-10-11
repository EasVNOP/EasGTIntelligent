# CPM-Bee Finetuning Example with CCPM

1. 准备原始数据，格式如下：
```json
{"target": "3", "input": "[翻译]昏暗的灯熄灭了又被重新点亮。[0]渔灯灭复明[1]残灯灭又然[2]残灯暗复明[3]残灯灭又明[答案]"}
```
放置在路径`src/ccpm_example/raw_data/`下

准备模型checkpoint，放在路径`src/ckpts/pytorch_model.bin`下

2. 进入工作路径
```bash
$ cd src
```

2. 重新调整数据格式
```bash
$ python ccpm_example/data_reformat.py
```
得到格式
```json
{"input": "昏暗的灯熄灭了又被重新点亮。", "options": {"<option_0>": "渔灯灭复明", "<option_1>": "残灯灭又然", "<option_2>": "残灯暗复明", "<option_3>": "残灯灭又明"}, "question": "这段话形容了哪句诗的意境？", "<ans>": "<option_3>"}
```
放置在路径`src/ccpm_example/bee_data/`下

注：该格式为参考格式。微调时，您可以自由设计您的数据格式，可以不设置"prompt"字段，只要所提供的数据涵盖所有必要信息即可，但我们一般推荐将输入文本字段标识为"input"/"document"/"doc"，如果是选择题，则应当添加"options"字段与"question"字段；如果是一般的文本生成，包含"input"+"\<ans\>"即可

3. 构建二进制数据文件
```bash
$ python preprocess_dataset.py --input ccpm_example/bee_data --output_path ccpm_example/bin_data --output_name ccpm_data
```
放在路径`ccpm_example/bin_data/`下

注：应确保没有同名路径`ccpm_example/bin_data/`，如存在同名路径，应先删除该路径再运行上述指令。如未提前删除，该指令会报错`ValueError: Dataset name exists`，同时产生一个新路径`tmp/`，此时应当连同`tmp/`与同名路径`ccpm_example/bin_data/`一并删除，之后再运行上述指令即可。

4. 修改模型微调脚本scripts/finetune_cpm_bee.sh为：
```bash
#! /bin/bash
# 四卡微调
export CUDA_VISIBLE_DEVICES=0,1,2,3
GPUS_PER_NODE=4

NNODES=1
MASTER_ADDR="localhost"
MASTER_PORT=12346

OPTS=""
OPTS+=" --use-delta"  # 使用delta-tuning
OPTS+=" --model-config config/cpm-bee-10b.json"  # 模型配置文件
OPTS+=" --dataset ccpm_example/bin_data/train"  # 训练集路径
OPTS+=" --eval_dataset ccpm_example/bin_data/train"  # 验证集路径
OPTS+=" --epoch 5"  # 训练epoch数
OPTS+=" --batch-size 5"
OPTS+=" --train-iters 100"  # 用于lr_schedular
OPTS+=" --save-name cpm_bee_finetune"  # 保存名称
OPTS+=" --max-length 2048"
OPTS+=" --save results/"  # 保存路径
OPTS+=" --lr 0.0001"
OPTS+=" --inspect-iters 100"
OPTS+=" --warmup-iters 1"
OPTS+=" --eval-interval 50"  # 每50步验证一次
OPTS+=" --early-stop-patience 5"  # 如果验证集loss连续5次不降，停止微调
OPTS+=" --lr-decay-style noam"
OPTS+=" --weight-decay 0.01"
OPTS+=" --clip-grad 1.0"
OPTS+=" --loss-scale 32768"
OPTS+=" --start-step 0"
OPTS+=" --load ckpts/pytorch_model.bin"  # 模型参数文件

CMD="torchrun --nnodes=${NNODES} --nproc_per_node=${GPUS_PER_NODE} --rdzv_id=1 --rdzv_backend=c10d --rdzv_endpoint=${MASTER_ADDR}:${MASTER_PORT} finetune_cpm_bee.py ${OPTS}"

echo ${CMD}
$CMD
```

5. 开始微调
```bash
bash scripts/finetune_cpm_bee.sh
```
您可以在`src/results/`中查看存储的delta模型