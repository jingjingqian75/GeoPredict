# Training on RoboCasa

## 1. Prepare the Environment

Follow the [environment setup instructions](infer.md#2-create-conda-environment) to create the `geopredict` conda environment and install the required dependencies.

Install the additional training dependencies:

```bash
conda activate geopredict
cd /path/to/GeoPredict
python -m pip install deepspeed opencv-python
```

Install the [Gaussian rasterizer with depth and alpha support](https://github.com/ashawkey/diff-gaussian-rasterization). The following command uses the same revision as our training environment:

```bash
python -m pip install --no-build-isolation "git+https://github.com/ashawkey/diff-gaussian-rasterization.git@8829d14f814fccdaf840b7b0f3021a616583c0a1"
```

Building the rasterizer requires a CUDA toolkit compatible with the installed PyTorch and a C++ compiler.

Use CUDA GPUs with BF16 support, as the training script and DeepSpeed configuration use BF16 precision.

## 2. Download Checkpoints and Dataset

Download the pretrained Pi0 weights (`pi0_base.pth`), `paligemma_tokenizer.model`, and `robocasa_norm_stats.json` from the [GeoPredict-Robocasa Hugging Face repository](https://huggingface.co/Jingjing0601/GeoPredict-Robocasa). Save all three files into the `ckpts/` folder under the project root.

Download the training dataset from the [GeoPredict-Robocasa ModelScope repository](https://modelscope.cn/datasets/JingjingQian/GeoPredict-Robocasa). Extract any downloaded archives and arrange the files under `data/robocasa/`. The expected file structure is:

```text
GeoPredict/
├── ckpts/
│   ├── pi0_base.pth
│   ├── paligemma_tokenizer.model
│   └── robocasa_norm_stats.json
├── data/
│   └── robocasa/
│       ├── data/
│       │   └── <episode_name>/
│       │       ├── infos.npy
│       │       ├── keypoints.npy
│       │       ├── cams.npy
│       │       ├── agentview_left_image/
│       │       │   └── step_0000.png
│       │       ├── agentview_right_image/
│       │       │   └── step_0000.png
│       │       ├── eye_in_hand_image/
│       │       │   └── step_0000.png
│       │       ├── agentview_left_depth/
│       │       │   └── step_0000.npy
│       │       └── agentview_right_depth/
│       │           └── step_0000.npy
│       └── meta/
│           └── episodes.json
├── configs/
│   └── ds_bf16_z3_config.json
├── tools/
│   └── train_robocasa.py
└── train_robocasa.sh
```

Each episode contains images and depth maps for all steps, named `step_XXXX.png` and `step_XXXX.npy`. The `meta/episodes.json` file maps episode names to language instructions.

The training script reads data from `./data/robocasa` by default. To use another location, update `data_root` in `tools/train_robocasa.py`.

## 3. Run Training

After preparing the environment, checkpoints, and dataset, run the following command from the project root:

```bash
bash train_robocasa.sh
```

The script uses DeepSpeed ZeRO-3 with BF16 precision. The default training settings are:

| Setting | Value |
| --- | --- |
| Training steps | 40,000 |
| Optimizer | AdamW |
| Learning rate | 2.5e-5 |
| Warmup steps | 1,000 |
| Learning rate schedule | Cosine decay to 2.5e-6 at step 30,000 |
| Micro batch size per GPU | 4 |
| Gradient accumulation steps | 1 |
| Logging interval | 100 steps |
| Checkpoint interval | 5,000 steps |

Edit `train_robocasa.sh` to change the experiment name, pretrained checkpoint path, or training steps. Additional arguments are defined in `utils/arguments.py`.

To change the batch size, edit `train_micro_batch_size_per_gpu` and `gradient_accumulation_steps` in `configs/ds_bf16_z3_config.json`. The `--batch_size` argument alone does not change the DeepSpeed dataloader batch size. The effective batch size is the micro batch size per GPU multiplied by the number of GPUs and gradient accumulation steps.

## 4. Training Outputs

Logs and EMA model checkpoints are saved under `experiments/`:

```text
experiments/
└── <timestamp>_geopredict_robocasa/
    ├── log.txt
    └── checkpoints/
        ├── checkpoint_step_5000.pth
        ├── ...
        ├── checkpoint_step_40000.pth
        └── final_checkpoint.pth
```

To evaluate a trained checkpoint, set `--weight_path` in `test_robocasa.sh` to the saved checkpoint and follow the [evaluation instructions](infer.md#3-run-evaluation).
