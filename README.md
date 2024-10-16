# DIAL-MPC: Diffusion-Inspired Annealing For Legged MPC

<div align="center">

[[Website]](https://anonymous-dial-mpc.github.io/dial-mpc/)

[<img src="https://img.shields.io/badge/Backend-Jax-red.svg"/>](https://github.com/google/jax)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

<img src="assets/joint.gif" width="600px"/>

</div>

This repository contains the code (simulation and real-world experiments with minimum setup) for the paper "Full-Order Sampling-Based MPC for Torque-Level Locomotion Control via Diffusion-Style Annealing".

DIAL-MPC is a sampling-based MPC framework for legged robot ***full-order torque-level*** control with both precision and agility in a ***training-free*** manner. 
DIAL-MPC is designed to be simple and flexible, with minimal requirements for specific reward design and dynamics model. It directly samples and rolls out in physics-based simulations (``Brax``) and does not require reduced-order modeling, linearization, convexification, or predefined contact sequences.
That means you can test out the controller in a plug-and-play manner with minimum setup.

## Simulation Setup

### Install `dial-mpc`

> [!IMPORTANT]
> We recommend Ubuntu >= 20.04 + Python >= 3.10 + CUDA >= 12.3.
> You can create a mamba (or conda) environment before proceeding.

Our environment is Ubuntu 22.04 + Python 3.10 + CUDA 12.6.

```bash
git clone https://github.com/anonymous-dial-mpc/dial-mpc.git --depth 1
cd dial-mpc
pip3 install -e .
```

## Synchronous Simulation

#### Run Examples

List available examples:

```bash
dial-mpc --list-examples
```

Run an example:

```bash
dial-mpc --example unitree_h1_jog
```

After rollout completes, go to `127.0.0.1:5000` to visualize the rollouts.

## Asynchronous Simulation

The asynchronous simulation is meant to test the algorithm before Sim2Real.

List available examples:

```bash
dial-mpc-sim --list-examples
```

Run an example:

In terminal 1, run

```bash
dial-mpc-sim --example unitree_go2_seq_jump_deploy
```
This will open a mujoco visualization window.

In terminal 2, run

```bash
dial-mpc-plan --example unitree_go2_seq_jump_deploy
```


## Deploy in Real

🚧 Check back in late Sep. - early Oct. 2024 for real-world deployment pipeline on Unitree Go2.
<!-- ### Install `unitree_sdk2_python`
Execute the following commands in the terminal:

```bash
cd ~
sudo apt install python3-pip
git clone https://github.com/unitreerobotics/unitree_sdk2_python.git
cd unitree_sdk2_python
pip3 install -e .
``` -->

## Writing Custom Environment

1. If custom robot model is needed, Store it in `dial_mpc/models/my_model/my_model.xml`.
2. Import the base environment and config.
3. Implement required functions.
4. Register environment.
5. Configure config file.

Example environment file (`my_env.py`):

```python
from dataclasses import dataclass

from brax import envs as brax_envs
from brax.envs.base import State

from dial_mpc.envs.base_env import BaseEnv, BaseEnvConfig
import dial_mpc.envs as dial_envs

@dataclass
class MyEnvConfig(BaseEnvConfig):
    arg1: 1.0
    arg2: "test"

class MyEnv(BaseEnv):
    def __init__(self, config: MyEnvConfig):
        super().__init__(config)
        # custom initializations below...

    def make_system(self, config: MyEnvConfig) -> System:
        model_path = ("my_model/my_model.xml")
        sys = mjcf.load(model_path)
        sys = sys.tree_replace({"opt.timestep": config.timestep})
        return sys

    def reset(self, rng: jax.Array) -> State:
        # TODO: implement reset

    def step(self, state: State, action: jax.Array) -> State:
        # TODO: implement step

brax_envs.register_environment("my_env_name", MyEnv)
dial_envs.register_config("my_env_name", MyEnvConfig)
```

Example configuration file (`my_env.yaml`):
```yaml
# DIAL-MPC
seed: 0
output_dir: dial_mpc_ws/my_model
n_steps: 400

env_name: my_env_name
Nsample: 2048
Hsample: 25
Hnode: 5
Ndiffuse: 4
Ndiffuse_init: 10
temp_sample: 0.05
horizon_diffuse_factor: 1.0
traj_diffuse_factor: 0.5
update_method: mppi


# Base environment
dt: 0.02
timestep: 0.02
leg_control: torque
action_scale: 1.0

# My Env
arg1: 2.0
arg2: "test_2"
```

Run the following command to use the custom environment in synchronous simulation. Make sure that `my_env.py` is in the same directory from which the command is run.

```bash
dial-mpc --config my_env.yaml --custom-env my_env
```

You can also run asynchronous simulation with the custom environment:

```bash
# Terminal 1
dial-mpc-sim --config my_env.yaml --custom-env my_env

# Terminal 2
dial-mpc-plan --config my_env.yaml --custom-env my_env
```

## Acknowledgements

* This codebase's environment and RL implementation is built on top of [Brax](https://github.com/google/brax).
* We use [Mujoco MJX](https://github.com/deepmind/mujoco) for the physics engine.
* Controller design and implementation is inspired by [Model-based Diffusion](https://github.com/LeCAR-Lab/model-based-diffusion).
