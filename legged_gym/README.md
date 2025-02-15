# Robot Parkour Learning (Tutorial) #
This is the tutorial for training the skill policy and distilling the parkour policy.

## Installation ##
1. Create a new Python virtual env or conda environment with Python 3.6, 3.7, or 3.8 (3.8 recommended)
2. Install PyTorch 1.10 with cuda-11.3:
    - `pip install torch==1.10.0+cu113 torchvision==0.11.1+cu113 torchaudio==0.10.0+cu113 -f https://download.pytorch.org/whl/cu113/torch_stable.html`
3. Install Isaac Gym
   - Download and install Isaac Gym Preview 4 (I didn't test the history version) from https://developer.nvidia.com/isaac-gym
   - `cd isaacgym/python && pip install -e .`
   - Try running an example `cd examples && python 1080_balls_of_solitude.py`
   - For troubleshooting check docs `isaacgym/docs/index.html`
4. Install rsl_rl (PPO implementation)
   - Using the command to direct to the root path of this repository
   - `cd rsl_rl && pip install -e .` 
5. Install legged_gym
   - `cd ../legged_gym && pip install -e .`

## Usage ##
***Always run your script in the root path of this legged_gym folder (which contains a `setup.py` file).***

1. The specialized skill policy is trained using `a1_field_config.py` as task `a1_field`

    Run command with `python legged_gym/scripts/train.py --headless --task a1_field`
    
2. The distillation is done using `a1_field_distill_config.py` as task `a1_distill`

    The distillation is done in multiprocess. In general, you need at least 2 processes, each with 1 GPU, and can access a shared folder.

    With `python legged/scripts/train.py --headless --task a1_distill` you launch the trainer. The process will prompt you to launch a collector process, where the log directory is corresponding to the task.

    With `python legged/scripts/collect.py --headless --task a1_distill --load_run {your training run}` you lauched the collector. The process will load the training policy and start collecting the data. The collected data will be saved in the directory prompted by the trainer. Remove it after you finish distillation.

### Train a walk policy ###

Launch the training by `python legged_gym/scripts/train.py --headless --task a1_field`. You will find the training log in `logs/a1_field`. The folder name is also the run name.

### Train each separate skill ###

- Launch the scirpt with task `a1_climb`, `a1_leap`, `a1_crawl`, `a1_tilt`. The training log will also be saved in `logs/a1_field`.

- The default training is in soft-dynamics constraint. To train in hard-dynamics constraint, change the `virtural_terraion` value to `False` in the corresponding config file.

    - Uncomment the `class sensor` part to enable the proprioception delay.

    - For `a1_climb`, please update the `climb["depth"]` field when switching to hard-dynamics constraint.

- Do remember to update the `load_run` field in the corresponding log directory to load the policy from the previous stage.

### Distill the parkour policy ###

**You will need at least two GPUs that can render in IsaacGym and have at least 24GB of memory. (typically RTX 3090)**

1. Update the `A1FieldDistillCfgPPO.algorithm.teacher_policy.sub_policy_paths` field with the logdir of your own trained skill policy. (in `a1_field_distill_config.py`)

2. Run data collection using `python legged_gym/scripts/collect.py --headless --task a1_distill`. The data will be saved in the `logs/distill_{robot}_dagger` directory.

    ***You can generate multiple datasets by running this step multiple times.***

    ***You can use soft link to put the `logs/distill_{robot}_dagger` directoy to a faster filesystem to speed up the training and data-collecting process.***

3. Update the `A1FieldDistillCfgPPO.runner.pretrain_dataset.data_dir` field with a list of dataset directories. Comment out the `A1FieldDistillCfgPPO.runner.pretrain_dataset.scan_dir` field. (in `a1_field_distill_config.py`)

4. Run `python legged_gym/scripts/train.py --headless --task a1_distill` to start distillation. The distillation log will be saved in `logs/distill_{robot}`.

5. Comment out `A1FieldDistillCfgPPO.runner.pretrain_dataset.data_dir` field and uncomment `A1FieldDistillCfgPPO.runner.pretrain_dataset.scan_dir` field. (in `a1_field_distill_config.py`)

6. Update the `A1FieldDistillCfgPPO.runner.load_run` field with your last distillation log.

7. Run `python legged_gym/scripts/train.py --headless --task a1_distill` to start dagger. The terminal will prompt you to launch a collector process.

    To run the collector process, change `RunnerCls` to `DaggerSaver` (as in lines 20-21). Run `python legged_gym/scripts/collect.py --headless --task a1_distill --load_run {the training run prompt by the training process}`.

    ***You can run multiple collector processes to speed up the data collection. And follow the options comments in the collector script.***

    ***The collectors can be run on any other machine as long as the `logs/distill_{robot}_dagger` and `logs/distill_{robot}` directory is shared among them.***

    Press Enter in the training process when you see some data is collected (prompted by the collector process). The training process will load the collected data and start training.

### Visualize the policy in simulation ###

```bash
python legged_gym/scripts/play.py --task {task} --load_run {run_name}
```
