# 复现笔记

![](images/teaser.jpeg)

## 配置
> Isaac Gym 版本/依赖来自 `legged_gym/README.md`，这份代码基于 Isaac Gym Preview 4。

```sh
# 1) 安装 Isaac Gym (Preview 4)
# 下载/解压后
cd isaacgym/python && pip install -e .

# 2) 安装 rsl_rl
cd /workspace/parkour/rsl_rl && pip install -e .

# 3) 安装 legged_gym
cd /workspace/parkour/legged_gym && pip install -e .
```

**注意**：`train.py/play.py/collect.py` 都要求在 `legged_gym/` 目录下运行（见 `legged_gym/README.md`）。

## 运行所需文件（train / play / collect）
> 只列本仓库关键入口和配置文件。

### 入口脚本（训练 / 可视化 / 数据采集）
- 训练入口：`legged_gym/legged_gym/scripts/train.py`
- 推理/可视化：`legged_gym/legged_gym/scripts/play.py`
- 轨迹采集（DAgger/演示）：`legged_gym/legged_gym/scripts/collect.py`
- 清理采集数据：`legged_gym/legged_gym/scripts/clear_dataset.py`

### Task/Env 注册与配置
- 任务注册：`legged_gym/legged_gym/envs/__init__.py`
  - 已注册：`go2`, `go2_field`, `go2_distill`, `go1_field`, `go1_distill`, `a1`, `a1_remote`, `go1_remote` 等。
  - **A1 蒸馏配置存在但未注册**：`legged_gym/legged_gym/envs/a1/a1_field_distill_config.py`，若要 `--task a1_distill` 需自行注册。
- Go2 训练配置：
  - 走路：`legged_gym/legged_gym/envs/go2/go2_config.py`
  - Parkour Teacher：`legged_gym/legged_gym/envs/go2/go2_field_config.py`
  - Parkour Student（深度蒸馏）：`legged_gym/legged_gym/envs/go2/go2_distill_config.py`
- A1/Go1 相关配置：`legged_gym/legged_gym/envs/a1/**`、`legged_gym/legged_gym/envs/go1/**`

### Env/MDP 实现
- 环境基类：`legged_gym/legged_gym/envs/base/legged_robot.py`
- Parkour/障碍扩展：`legged_gym/legged_gym/envs/base/legged_robot_field.py`
- 传感器延迟 + 深度噪声：`legged_gym/legged_gym/envs/base/legged_robot_noisy.py`
- 地形生成（BarrierTrack/Perlin）：`legged_gym/legged_gym/utils/terrain/**`

### 算法/网络
- PPO/蒸馏：`rsl_rl/rsl_rl/algorithms/ppo.py`, `rsl_rl/rsl_rl/algorithms/tppo.py`, `rsl_rl/rsl_rl/algorithms/estimator.py`
- 两阶段训练（先数据集、后在线）：`rsl_rl/rsl_rl/runners/two_stage_runner.py`
- DAgger 数据采集：`rsl_rl/rsl_rl/runners/dagger_saver.py`
- 视觉编码器/Actor：`rsl_rl/rsl_rl/modules/encoder_actor_critic.py`, `rsl_rl/rsl_rl/modules/conv2d.py`

### 真实机器人部署
- Go1 部署：`onboard_codes/Deploy-Go1.md`
- Go2 部署：`onboard_codes/Deploy-Go2.md`
- 深度输入与编码（机载）：`onboard_codes/go1/go1_visual_embedding.py`, `onboard_codes/go2/go2_visual.py`

### 预训练 / 权重
- Go1 预训练压缩包：`go1_ckpts/*.zip`
- 训练日志默认路径：`legged_gym/logs/<experiment_name>/<timestamp>_<run_name>/model_*.pt`

## 直接看预训练结果（示例）
`go1_ckpts/` 里提供了 Go1 的 distill/walk 训练包，解压到 `legged_gym/logs/` 后即可 `play.py`：

```sh
# 例：解压到 logs/ 后
cd /workspace/parkour/legged_gym
unzip ../go1_ckpts/Nov02_16-18-16_674k_distill_crawljumpjumpleap_*.zip -d logs/

# 可视化推理
python legged_gym/scripts/play.py --task go1_distill --load_run <解压后目录名>
```

**注意**：README 提醒 distill policy 在仿真 play 时需要设置 `ckpt_manipulator = None`（见 `README.md`），否则可能加载不对。

## 重新训练（Go2 三阶段示例）
1. **阶段 0：基础行走（Go2Rough）**
   ```sh
   cd /workspace/parkour/legged_gym
   python legged_gym/scripts/train.py --headless --task go2
   ```
   日志：`legged_gym/logs/rough_go2/<timestamp>_Go2Rough*/model_*.pt`

2. **阶段 1：Oracle Parkour（Teacher / Go2Field）**
   - 先把 `go2_field_config.py` 里的 `load_run` 指向上一步的 walking 日志目录。
   ```sh
   python legged_gym/scripts/train.py --headless --task go2_field
   ```
   日志：`legged_gym/logs/field_go2/<timestamp>_Go2_*/model_*.pt`

3. **阶段 2：Student 蒸馏（Go2Distill）**
   - 先在 `go2_distill_config.py` 中填好：
     - `teacher_ac_path`：Oracle 模型路径
     - `pretrain_dataset.data_dir`：采集数据目录（多进程时）
   ```sh
   # 训练进程（多卡可配合 collect.py）
   python legged_gym/scripts/train.py --headless --task go2_distill

   # 采集进程（如果 multi_process_=True）
   python legged_gym/scripts/collect.py --headless --task go2_distill --log --load_run <训练进程提示的run>
   ```

4. **评估/可视化**
   ```sh
   python legged_gym/scripts/play.py --task go2_distill --load_run <run_name>
   ```

## 代码理解
> 重点拆解 Go2 Teacher/Student 两个阶段；A1/Go1 与其同源，差异点见后文。

### 阶段 0：Go2Rough（基础走路）
**任务与环境**
- 任务：`go2`（`legged_gym/legged_gym/envs/__init__.py`）
- 配置：`legged_gym/legged_gym/envs/go2/go2_config.py`

**观测（状态）与维度**（`legged_gym/legged_gym/envs/base/legged_robot.py`）
- `lin_vel` 3（Go2 设置 `use_lin_vel=False`，actor 观测里为 0）
- `ang_vel` 3
- `projected_gravity` 3
- `commands` 3
- `dof_pos` 12
- `dof_vel` 12
- `last_actions` 12
- `height_measurements` 1×21×11=231（`go2_config.py`：测点 x=21, y=11）
- **总维度**：48 + 231 = 279

**动作**
- 12 维关节位置增量（P 控制），`action_scale=0.5`（`go2_config.py`）。

**奖励（权重）**（`go2_config.py`）
- `tracking_lin_vel`: +1
- `tracking_ang_vel`: +1
- `energy_substeps`: -2e-5
- `stand_still`: -2
- `dof_error_named`: -1
- `dof_error`: -0.01
- `exceed_dof_pos_limits`: -0.4
- `exceed_torque_limits_l1norm`: -0.4
- `dof_vel_limits`: -0.4

### 阶段 1：Teacher / Oracle Parkour（Go2Field）
**任务与环境**
- 任务：`go2_field`（`legged_gym/legged_gym/envs/__init__.py`）
- 地形：BarrierTrack（台阶、坡、跳跃等），`go2_field_config.py`
- 环境类：`RobotFieldNoisy`（传感器延迟/噪声；`legged_gym/legged_gym/envs/base/robot_field_noisy.py`）

**观测与维度**
- 与阶段 0 相同：`48 + height_measurements(231) = 279`。
- `height_measurements` 仍是 21×11（`go2_config.py`）。

**动作**
- 12 维关节位置增量（同阶段 0）。

**奖励（权重）**（`go2_field_config.py`）
- `tracking_lin_vel`: +1
- `tracking_ang_vel`: +1
- `energy_substeps`: -2e-7
- `torques`: -1e-7
- `stand_still`: -1
- `dof_error_named`: -1
- `dof_error`: -0.005
- `collision`: -0.05
- `lazy_stop`: -3
- `exceed_dof_pos_limits`: -0.1
- `exceed_torque_limits_l1norm`: -0.1
- `penetrate_depth`: -0.05（`legged_robot_field.py`）

**Teacher 网络与算法**
- Policy：`EncoderStateAcRecurrent`（GRU）
- Height encoder：`MlpModel` 输出 32 维 latent（`go2_config.py` + `rsl_rl/rsl_rl/modules/encoder_actor_critic.py`）
- 估计器：`EstimatorPPO`（`rsl_rl/rsl_rl/algorithms/estimator.py`），将 `ang_vel/projected_gravity/commands/dof_pos/dof_vel/last_actions` 估计 `lin_vel`。

### 阶段 2：Student 蒸馏（Go2Distill）
**任务与环境**
- 任务：`go2_distill`（`legged_gym/legged_gym/envs/__init__.py`）
- 观测切到 `forward_depth`，privileged 仍保留 `height_measurements`。

**观测与维度**
- Actor obs：
  - `lin_vel` 3（Go2 仍 `use_lin_vel=False`，actor 观测中为 0）
  - `ang_vel` 3
  - `projected_gravity` 3
  - `commands` 3
  - `dof_pos` 12
  - `dof_vel` 12
  - `last_actions` 12
  - `forward_depth` 1×48×64 = 3072
  - **总维度**：48 + 3072 = 3120
- Critic/Teacher obs：`48 + height_measurements(231) = 279`

**动作**
- 12 维关节位置增量（同 Teacher）。

**奖励**
- 继承 Go2Field（同阶段 1 的 reward scales）。

**蒸馏算法与网络**
- 算法：`EstimatorTPPO`（`rsl_rl/rsl_rl/algorithms/estimator.py`）
  - `using_ppo = False`：主要用蒸馏损失训练（`rsl_rl/rsl_rl/algorithms/tppo.py`）
  - `distill_target = l1`（动作 L1）
- Teacher policy：`EncoderStateAcRecurrent`，权重从 `teacher_ac_path` 加载（`go2_distill_config.py`）
- Student 视觉编码器：`Conv2dHeadModel`（通道 [16,32,32], kernel [5,4,3], stride [2,2,1], maxpool）→ latent 32 维（`go2_distill_config.py`）
- 两阶段训练：
  - `TwoStageRunner` 支持先用数据集预训练，再 online（`rsl_rl/rsl_rl/runners/two_stage_runner.py`）
  - `collect.py` 用 `DaggerSaver` 采集 distill 数据（`rsl_rl/rsl_rl/runners/dagger_saver.py`）

### 蒸馏阶段的深度噪声（Sim2Real 关键）
> **重点**：Student 深度图不是直接用 Isaac Gym 输出，而是经过一套“类 RealSense D435i”的噪声与延迟模拟。

**1) 噪声模拟入口与处理顺序**
- 处理函数：`legged_gym/legged_gym/envs/base/legged_robot_noisy.py` 的 `_process_depth_image()`
- 流程：
  1. 反转深度（IsaacGym 深度为负值）
  2. 立体相机噪声：`_add_depth_stereo()`
  3. 远景/天空伪影：`_add_sky_artifacts()`
  4. 裁剪 `depth_range` → 归一化到 (0,1)
  5. 按 `crop_*` 裁剪，再 resize 到 `output_resolution`

**2) Stereo 噪声（Go2Distill 参数）**（`go2_distill_config.py`）
- `stereo_min_distance=0.175`：过近点处理
- `stereo_far_distance=1.2`：过远点处理
- `stereo_far_noise_std=0.08`、`stereo_near_noise_std=0.02`：近/远随机噪声
- `stereo_full_block_artifacts_prob=0.008` + `stereo_full_block_values=[0.0,0.25,0.5,1.,3.]`
  - 过近区域随机整块失真
- `stereo_half_block_spark_prob=0.02` + `stereo_half_block_value=3000`
  - 过近区域随机火花噪声

**3) Sky 伪影（Go2Distill 参数）**
- `sky_artifacts_prob=0.0001`
- `sky_artifacts_far_distance=2.0`
- `sky_artifacts_values=[0.6,1.,1.2,1.5,1.8]`

**4) 相机位姿/延迟随机化（Go2Distill）**
- 位姿随机：`forward_camera.position.mean/std`、`rotation.lower/upper`
- 延迟：`latency_range=[0.08,0.142]`，`refresh_duration=0.1s`
- 裁剪与分辨率：`crop_top_bottom`, `crop_left_right`, `output_resolution=48x64`

**5) A1Distill 的深度噪声参数（对比）**
- 路径：`legged_gym/legged_gym/envs/a1/a1_field_distill_config.py`
- 关键区别：`depth_range=1.5m`、`latency_range=0.2~0.26s`、`stereo_min_distance=0.12`

> **结论**：深度噪声 + 相机位姿/延迟抖动 + 立体相机伪影模拟构成了 sim2real 的核心实现，代码集中在 `legged_gym/legged_gym/envs/base/legged_robot_noisy.py` 与 `*_distill_config.py` 中。

### Sim2Real 实现细化（代码级）
> 下面按“配置 → 环境代码 → 部署代码”的链路，把 sim2real 的关键机制全部串起来。每个小节都给出对应代码段落，便于直接改参数或定位逻辑。

#### 1) 动力学/执行器随机化（Domain Rand）
**配置（Go2 示例）**：`legged_gym/legged_gym/envs/go2/go2_config.py`
```python
class domain_rand( LeggedRobotCfg.domain_rand ):
    randomize_com = True
    class com_range:
        x = [-0.2, 0.2]
        y = [-0.1, 0.1]
        z = [-0.05, 0.05]

    randomize_motor = True
    leg_motor_strength_range = [0.8, 1.2]

    randomize_base_mass = True
    added_mass_range = [1.0, 3.0]

    randomize_friction = True
    friction_range = [0., 2.]

    init_base_pos_range = dict(
        x= [0.05, 0.6],
        y= [-0.25, 0.25],
    )
    init_base_rot_range = dict(
        roll= [-0.75, 0.75],
        pitch= [-0.75, 0.75],
    )
    init_base_vel_range = dict(
        x= [-0.2, 1.5],
        y= [-0.2, 0.2],
        z= [-0.2, 0.2],
        roll= [-1., 1.],
        pitch= [-1., 1.],
        yaw= [-1., 1.],
    )
    init_dof_vel_range = [-5, 5]

    push_robots = True 
    max_push_vel_xy = 0.5
    push_interval_s = 2
```

**应用到仿真（摩擦/质量/质心）**：`legged_gym/legged_gym/envs/base/legged_robot.py`
```python
def _process_rigid_shape_props(self, props, env_id):
    if self.cfg.domain_rand.randomize_friction:
        if env_id==0:
            friction_range = self.cfg.domain_rand.friction_range
            num_buckets = 64
            bucket_ids = torch.randint(0, num_buckets, (self.num_envs, 1))
            friction_buckets = torch_rand_float(friction_range[0], friction_range[1], (num_buckets,1), device='cpu')
            self.friction_coeffs = friction_buckets[bucket_ids]
        for s in range(len(props)):
            props[s].friction = self.friction_coeffs[env_id]
    return props

def _process_rigid_body_props(self, props, env_id):
    if self.cfg.domain_rand.randomize_base_mass:
        rng = self.cfg.domain_rand.added_mass_range
        props[0].mass += np.random.uniform(rng[0], rng[1])
    if getattr(self.cfg.domain_rand, "randomize_com", False):
        rng_com_x = self.cfg.domain_rand.com_range.x
        rng_com_y = self.cfg.domain_rand.com_range.y
        rng_com_z = self.cfg.domain_rand.com_range.z
        rand_com = np.random.uniform(
            [rng_com_x[0], rng_com_y[0], rng_com_z[0]],
            [rng_com_x[1], rng_com_y[1], rng_com_z[1]],
            size=(3,),
        )
        props[0].com += gymapi.Vec3(*rand_com)
    return props
```

**电机强度随机化与执行扭矩**：`legged_gym/legged_gym/envs/base/legged_robot.py`
```python
self.motor_strength = torch.ones(self.num_envs, self.num_actions, dtype=torch.float, device=self.device, requires_grad=False)
if getattr(self.cfg.domain_rand, "randomize_motor", False):
    mtr_rng = self.cfg.domain_rand.leg_motor_strength_range
    self.motor_strength = torch_rand_float(
        mtr_rng[0],
        mtr_rng[1],
        (self.num_envs, self.num_actions),
        device=self.device,
    )

def _compute_torques(self, actions):
    actions = self.motor_strength * actions
    actions_scaled = actions * self.cfg.control.action_scale
    torques = self.p_gains*(actions_scaled + self.default_dof_pos - self.dof_pos) - self.d_gains*self.dof_vel
    return torch.clip(torques, -self.torque_limits, self.torque_limits)
```

**外力推搡扰动**：`legged_gym/legged_gym/envs/base/legged_robot.py`
```python
if self.cfg.domain_rand.push_robots and  (self.common_step_counter % self.cfg.domain_rand.push_interval == 0):
    self._push_robots()

def _push_robots(self):
    max_vel = self.cfg.domain_rand.max_push_vel_xy
    self.root_states[:, 7:9] = torch_rand_float(-max_vel, max_vel, (self.num_envs, 2), device=self.device)
    self.gym.set_actor_root_state_tensor(self.sim, gymtorch.unwrap_tensor(self.all_root_states))
```

**初始状态随机化**：`legged_gym/legged_gym/envs/base/legged_robot.py`
```python
def _reset_dofs(self, env_ids):
    if getattr(self.cfg.domain_rand, "init_dof_pos_ratio_range", None) is not None:
        self.dof_pos[env_ids] = self.default_dof_pos * torch_rand_float(
            self.cfg.domain_rand.init_dof_pos_ratio_range[0],
            self.cfg.domain_rand.init_dof_pos_ratio_range[1],
            (len(env_ids), self.num_dof),
            device=self.device,
        )
    dof_vel_range = getattr(self.cfg.domain_rand, "init_dof_vel_range", [-3., 3.])
    self.dof_vel[env_ids] = torch.rand_like(self.dof_vel[env_ids]) * abs(dof_vel_range[1] - dof_vel_range[0]) + min(dof_vel_range)

def _reset_root_states(self, env_ids):
    if hasattr(self.cfg.domain_rand, "init_base_pos_range"):
        self.root_states[env_ids, 0:1] += torch_rand_float(*self.cfg.domain_rand.init_base_pos_range["x"], (len(env_ids), 1), device=self.device)
        self.root_states[env_ids, 1:2] += torch_rand_float(*self.cfg.domain_rand.init_base_pos_range["y"], (len(env_ids), 1), device=self.device)
```

#### 2) 动作延迟 + 传感器延迟
**动作延迟（可选开关）**：`legged_gym/legged_gym/envs/base/legged_robot_noisy.py`
```python
def pre_physics_step(self, actions):
    self.set_buffers_refreshed_to_false()
    return_ = super().pre_physics_step(actions)

    if isinstance(self.cfg.control.action_scale, (tuple, list)):
        self.cfg.control.action_scale = torch.tensor(self.cfg.control.action_scale, device= self.sim_device)
    if getattr(self.cfg.control, "computer_clip_torque", False):
        self.actions_scaled = self.actions * self.cfg.control.action_scale
        control_type = self.cfg.control.control_type
        if control_type == "P":
            actions_scaled_clipped = self.clip_position_action_by_torque_limit(self.actions_scaled)
        else:
            raise NotImplementedError
    else:
        actions_scaled_clipped = self.actions * self.cfg.control.action_scale

    if getattr(self.cfg.control, "action_delay", False):
        self.actions_history_buffer = torch.roll(self.actions_history_buffer, shifts= -1, dims= 0)
        self.actions_history_buffer[-1] = actions_scaled_clipped
        action_delayed_frames = ((self.action_delay_buffer / self.dt) + 1).to(int)
        self.actions_scaled_clipped = self.actions_history_buffer[
            -action_delayed_frames,
            torch.arange(self.num_envs, device= self.device),
        ]
    else:
        self.actions_scaled_clipped = actions_scaled_clipped

    return return_
```

**传感器延迟缓冲（含相机延迟）**：`legged_gym/legged_gym/envs/base/legged_robot_noisy.py`
```python
def set_latency_buffer_for_sensor(self, sensor_name):
    latency_buffer = torch_rand_float(
        getattr(self.cfg.sensor, sensor_name).latency_range[0],
        getattr(self.cfg.sensor, sensor_name).latency_range[1],
        (self.num_envs, 1),
        device= self.device,
    ).flatten()
    setattr(self, sensor_name + "_latency_buffer", latency_buffer)
    if "camera" in sensor_name:
        setattr(self, sensor_name + "_delayed_frames", torch.zeros_like(latency_buffer, dtype= torch.long, device= self.device))

def _resample_sensor_latency_if_needed(self, sensor_name):
    resampling_time = getattr(getattr(self.cfg.sensor, sensor_name), "latency_resampling_time", self.dt)
    resample_env_ids = (self.episode_length_buf % int(resampling_time / self.dt) == 0).nonzero(as_tuple= False).flatten()
    if len(resample_env_ids) > 0:
        getattr(self, sensor_name + "_latency_buffer")[resample_env_ids] = torch_rand_float(
            getattr(getattr(self.cfg.sensor, sensor_name), "latency_range")[0],
            getattr(getattr(self.cfg.sensor, sensor_name), "latency_range")[1],
            (len(resample_env_ids), 1),
            device= self.device,
        ).flatten()
```

**深度帧延迟刷新**：`legged_gym/legged_gym/envs/base/legged_robot_noisy.py`
```python
def _get_forward_depth_obs(self, privileged= False):
    if hasattr(self, "forward_depth_obs_buffer") and (not self.forward_depth_obs_refreshed) and hasattr(self.cfg.sensor, "forward_camera") and (not privileged):
        self.forward_depth_obs_buffer = torch.cat([
            self.forward_depth_obs_buffer[1:],
            self._process_depth_image(self.sensor_tensor_dict["forward_depth"]),
        ], dim= 0)
        delay_refresh_mask = (self.episode_length_buf % int(self.cfg.sensor.forward_camera.refresh_duration / self.dt)) == 0
        frame_select = (self.forward_camera_latency_buffer / self.dt).to(int)
        self.forward_camera_delayed_frames = torch.where(
            delay_refresh_mask,
            torch.minimum(frame_select, self.forward_camera_delayed_frames + 1),
            self.forward_camera_delayed_frames + 1,
        )
        self.forward_depth_output = self.forward_depth_obs_buffer[
            -self.forward_camera_delayed_frames,
            torch.arange(self.num_envs, device= self.device),
        ].clone()
```

**对应配置（Go2Distill 相机延迟）**：`legged_gym/legged_gym/envs/go2/go2_distill_config.py`
```python
class forward_camera:
    latency_range = [0.08, 0.142]
    latency_resampling_time = 5.0
    refresh_duration = 1/10
```

#### 3) 深度噪声/伪影（Sim2Real 核心）
**噪声处理入口与顺序**：`legged_gym/legged_gym/envs/base/legged_robot_noisy.py`
```python
@torch.no_grad()
def _process_depth_image(self, depth_images):
    depth_images_ = torch.stack(depth_images).unsqueeze(1).contiguous().detach().clone() * -1
    if hasattr(self.cfg.noise, "forward_depth"):
        if getattr(self.cfg.noise.forward_depth, "stereo_min_distance", 0.) > 0.:
            depth_images_ = self._add_depth_stereo(depth_images_)
        if getattr(self.cfg.noise.forward_depth, "sky_artifacts_prob", 0.) > 0.:
            depth_images_ = self._add_sky_artifacts(depth_images_)
    depth_images_ = self._normalize_depth_images(depth_images_)
    depth_images_ = self._crop_depth_images(depth_images_)
    if hasattr(self, "forward_depth_resize_transform"):
        depth_images_ = self.forward_depth_resize_transform(depth_images_)
    depth_images_ = depth_images_.clip(0, 1)
    return depth_images_.unsqueeze(0)
```

**Stereo 噪声与近距伪影**：`legged_gym/legged_gym/envs/base/legged_robot_noisy.py`
```python
def _add_depth_stereo(self, depth_images):
    N, _, H, W = depth_images.shape
    far_mask = depth_images > self.cfg.noise.forward_depth.stereo_far_distance
    too_close_mask = depth_images < self.cfg.noise.forward_depth.stereo_min_distance
    near_mask = (~far_mask) & (~too_close_mask)

    far_noise = torch_rand_float(0., self.cfg.noise.forward_depth.stereo_far_noise_std, (N, H * W), device= self.device).view(N, 1, H, W)
    near_noise = torch_rand_float(0., self.cfg.noise.forward_depth.stereo_near_noise_std, (N, H * W), device= self.device).view(N, 1, H, W)
    depth_images += far_noise * far_mask
    depth_images += near_noise * near_mask

    vertical_block_mask = self._recognize_top_down_too_close(too_close_mask)
    full_block_mask = vertical_block_mask & too_close_mask
    half_block_mask = (~vertical_block_mask) & too_close_mask
    for pixel_value in random.sample(self.cfg.noise.forward_depth.stereo_full_block_values, len(self.cfg.noise.forward_depth.stereo_full_block_values)):
        artifacts_buffer = torch.ones_like(depth_images)
        artifacts_buffer = self._add_depth_artifacts(artifacts_buffer,
            self.cfg.noise.forward_depth.stereo_full_block_artifacts_prob,
            self.cfg.noise.forward_depth.stereo_full_block_height_mean_std,
            self.cfg.noise.forward_depth.stereo_full_block_width_mean_std,
        )
        depth_images[full_block_mask] = ((1 - artifacts_buffer) * pixel_value)[full_block_mask]
    half_block_spark = torch_rand_float(0., 1., (N, H * W), device= self.device).view(N, 1, H, W) < self.cfg.noise.forward_depth.stereo_half_block_spark_prob
    depth_images[half_block_mask] = (half_block_spark.to(torch.float32) * self.cfg.noise.forward_depth.stereo_half_block_value)[half_block_mask]
    return depth_images
```

**天空/远景伪影**：`legged_gym/legged_gym/envs/base/legged_robot_noisy.py`
```python
def _add_sky_artifacts(self, depth_images):
    N, _, H, W = depth_images.shape
    possible_to_sky_mask = depth_images > self.cfg.noise.forward_depth.sky_artifacts_far_distance
    to_sky_mask = self._recognize_top_down_seeing_sky(possible_to_sky_mask)
    isinf_mask = depth_images.isinf()
    for pixel_value in random.sample(self.cfg.noise.forward_depth.sky_artifacts_values, len(self.cfg.noise.forward_depth.sky_artifacts_values)):
        artifacts_buffer = torch.ones_like(depth_images)
        artifacts_buffer = self._add_depth_artifacts(artifacts_buffer,
            self.cfg.noise.forward_depth.sky_artifacts_prob,
            self.cfg.noise.forward_depth.sky_artifacts_height_mean_std,
            self.cfg.noise.forward_depth.sky_artifacts_width_mean_std,
        )
        depth_images[to_sky_mask & (~isinf_mask)] *= artifacts_buffer[to_sky_mask & (~isinf_mask)]
        depth_images[to_sky_mask & isinf_mask & (artifacts_buffer < 1)] = 0.
        depth_images[to_sky_mask] += ((1 - artifacts_buffer) * pixel_value)[to_sky_mask]
    return depth_images
```

**这些噪声/伪影分别解决的现实问题**：
- `stereo_min_distance` / `_add_depth_stereo`：处理双目深度量程边界与噪声分布差异；远处增加随机噪声模拟视差量化/匹配不稳，近处增加噪声模拟近距测量抖动；“too_close” 触发整列/半列块状无效值（`full_block/half_block`），对齐 D435i 近距失效的条带/闪烁模式。
- `sky_artifacts_prob` / `_add_sky_artifacts`：针对天空/天花板/远景深度不可测或无限大的情况，检测“上方全是远景”区域并注入伪影或置零，模拟真实场景中 sky/ceiling 导致的大面积无效深度。
**是否仅限四足**：不是，这些问题是 D435i 这类双目深度相机的通用物理/算法限制，装在无人机上同样存在。区别在于场景与运动方式导致出现频率不同：无人机更容易出现快速姿态变化、振动与强光干扰，导致远距噪声和 sky/ceiling 伪影更常见；近距条带失效与 `stereo_min_distance` 则取决于前视角度与飞行高度。

#### 4) 相机外参/裁剪/分辨率对齐（Sim2Real 关键）
**配置（Go2Distill）**：`legged_gym/legged_gym/envs/go2/go2_distill_config.py`
```python
class forward_camera:
    resolution = [int(480/4), int(640/4)]
    position = dict(
        mean= [0.24, -0.0175, 0.12],
        std= [0.01, 0.0025, 0.03],
    )
    rotation = dict(
        lower= [-0.1, 0.37, -0.1],
        upper= [0.1, 0.43, 0.1],
    )
    resized_resolution = [48, 64]
    output_resolution = [48, 64]
    horizontal_fov = [86, 90]
    crop_top_bottom = [int(48/4), 0]
    crop_left_right = [int(28/4), int(36/4)]
    near_plane = 0.05
    depth_range = [0.0, 3.0]
```

#### 5) 观测噪声注入（可选）
**统一的 obs 噪声入口**：`legged_gym/legged_gym/envs/base/legged_robot.py`
```python
if self.add_noise == "uniform" or self.add_noise == True:
    self.obs_buf += (2 * torch.rand_like(self.obs_buf) - 1) * self.noise_scale_vec
elif self.add_noise == "gaussian":
    self.obs_buf += torch.randn_like(self.obs_buf) * self.noise_scale_vec
```

#### 6) IMU 重力偏置随机化（Go1 Distill）
**配置（Go1Distill）**：`legged_gym/legged_gym/envs/go1/go1_field_distill_config.py`
```python
class domain_rand( A1FieldDistillCfg.domain_rand ):
    randomize_gravity_bias = True
    randomize_privileged_gravity_bias = False
    gravity_bias_range = dict(
        x= [-0.12, 0.12],
        y= [-0.12, 0.12],
        z= [-0.05, 0.05],
    )

class noise( A1FieldDistillCfg.noise ):
    add_noise = True
    class noise_scales( A1FieldDistillCfg.noise.noise_scales ):
        ang_vel = 0.2
        dof_pos = 0.0006
        dof_vel = 0.02
        gravity = 0.06
```

**注入逻辑（偏置写入 proprioception）**：`legged_gym/legged_gym/envs/base/legged_robot_field.py`
```python
if getattr(self.cfg.domain_rand, "randomize_gravity_bias", False):
    self.gravity_bias = torch.rand(self.num_envs, 3, dtype= torch.float, device= self.device, requires_grad= False)
    self.gravity_bias[:, 0] *= self.cfg.domain_rand.gravity_bias_range["x"][1] - self.cfg.domain_rand.gravity_bias_range["x"][0]
    self.gravity_bias[:, 0] += self.cfg.domain_rand.gravity_bias_range["x"][0]
    self.gravity_bias[:, 1] *= self.cfg.domain_rand.gravity_bias_range["y"][1] - self.cfg.domain_rand.gravity_bias_range["y"][0]
    self.gravity_bias[:, 1] += self.cfg.domain_rand.gravity_bias_range["y"][0]
    self.gravity_bias[:, 2] *= self.cfg.domain_rand.gravity_bias_range["z"][1] - self.cfg.domain_rand.gravity_bias_range["z"][0]
    self.gravity_bias[:, 2] += self.cfg.domain_rand.gravity_bias_range["z"][0]

def _get_proprioception_obs(self, privileged= False):
    obs_buf = super()._get_proprioception_obs(privileged= privileged)
    if getattr(self.cfg.domain_rand, "randomize_gravity_bias", False) and (not privileged):
        proprioception_slice = get_obs_slice(self.obs_segments, "proprioception")
        obs_buf[:, proprioception_slice[0].start + 6: proprioception_slice[0].start + 9] += self.gravity_bias
    return obs_buf
```

#### 7) 地形随机化与感知挑战
**BarrierTrack 随机障碍 + Perlin 噪声**：`legged_gym/legged_gym/envs/go2/go2_field_config.py`
```python
class terrain( Go2RoughCfg.terrain ):
    selected = "BarrierTrack"
    BarrierTrack_kwargs = dict(
        options= [
            "jump", "leap", "hurdle", "down", "tilted_ramp",
            "stairsup", "stairsdown", "discrete_rect", "slope", "wave",
        ],
        add_perlin_noise= True,
        border_perlin_noise= True,
        virtual_terrain= False,
        draw_virtual_terrain= True,
        randomize_obstacle_order= True,
        n_obstacles_per_track= 1,
    )
```

**Perlin 参数（地表粗糙度）**：`legged_gym/legged_gym/envs/go2/go2_config.py`
```python
class terrain:
    TerrainPerlin_kwargs = dict(
        zScale= 0.07,
        frequency= 10,
    )
```

#### 8) 真机部署：深度预处理 + embedding 替换
**深度图对齐训练输入（Go2 RealSense 端）**：`onboard_codes/go2/go2_visual.py`
```python
depth_image_np = np.asanyarray(depth_frame.get_data())
depth_image_np = np.rot90(depth_image_np, k= 2)
depth_image_pyt = torch.from_numpy(depth_image_np.astype(np.float32)).unsqueeze(0).unsqueeze(0)
depth_image_pyt = depth_image_pyt[:, :,
    self.cropping[0]: -self.cropping[1]-1,
    self.cropping[2]: -self.cropping[3]-1,
]
depth_image_pyt = torch.clip(depth_image_pyt, self.depth_range[0], self.depth_range[1]) / (self.depth_range[1] - self.depth_range[0])
depth_image_pyt = resize2d(depth_image_pyt, self.output_resolution)
```

**用 embedding 替换 forward_depth 观测**：`onboard_codes/go2/go2_run.py`
```python
env_node = Go2Node(
    "go2",
    cfg= config_dict,
    replace_obs_with_embeddings= ["forward_depth"],
    model_device= device,
    dryrun= not args.nodryrun,
)
```

**接收 embedding 并写入 obs**：`onboard_codes/go2/unitree_ros2_real.py`
```python
def _forward_depth_embedding_callback(self, msg):
    self.forward_depth_embedding_buffer = torch.tensor(msg.data, device= self.model_device, dtype= torch.float32).view(1, -1)

def _get_forward_depth_embedding_obs(self):
    return self.forward_depth_embedding_buffer
```

> **指导建议**：如果仿真里调了 `output_resolution/crop/depth_range` 或 `latency_range`，务必同步到 `onboard_codes/go2/go2_visual.py` 的裁剪/归一化与发频设置，保证 real 输入与训练一致。

### A1 / Go1 任务补充（简要）
- A1 Field（Teacher/Skill）：`a1_field_config.py`
  - 观测：`proprioception(48) + base_pose(6) + robot_config(17) + engaging_block(203) + sidewall_distance(2)` → **总 276 维**
  - `engaging_block` 维度由 `BarrierTrack.max_track_options=200` 与 `block_info_dim=2` 决定（`barrier_track.py`）。
- A1 Distill：`a1_field_distill_config.py`
  - Actor obs：`proprioception(48) + forward_depth(48×64)` → **3120 维**
  - Privileged obs：同 A1 Field
  - Teacher 由 `ActorCriticClimbMutex` 合并多个技能策略（`rsl_rl/rsl_rl/modules/actor_critic_field_mutex.py`）
- Go1：配置见 `legged_gym/legged_gym/envs/go1/**`，部署脚本在 `onboard_codes/Deploy-Go1.md`

## 与 Isaaclab_Parkour 参考 note 的相同点 / 不同点
**相同点**
- 都是 Teacher → Student 的两阶段范式：Teacher 用高特权观测，Student 用深度图。
- 都有 forward depth 的视觉编码器 + 蒸馏损失，目标是 sim2real 部署。
- 都包含数据采集/回放/部署脚本与离线权重。

**不同点**
- 框架不同：这份代码基于 **Isaac Gym**（`legged_gym`），参考仓库基于 **IsaacLab**。
- 观测体系不同：这里用 `obs_components` 组合（`lin_vel/ang_vel/.../height_measurements/forward_depth`），IsaacLab 采用 MDP 配置拆分（prop/scan/priv 等）。
- 蒸馏算法不同：这里用 `TPPO/EstimatorTPPO + TwoStageRunner/DAgger`，参考仓库用自定义 `DistillationWithExtractor` 流程。
- 深度噪声实现更显式：本仓库在 `legged_robot_noisy.py` 中完整模拟立体相机噪声与伪影；参考仓库主要在传感器/观测配置里做扰动。
- 任务注册方式不同：这里依赖 `legged_gym/envs/__init__.py` 注册 task；参考仓库通过 IsaacLab 的 task registry 与 Hydra cfg。
