# Optimization of neural networks using weight sharing techniques

## Installation

You may install the dependencies with the following commands:

```sh
conda install pytorch pytorch-cuda=12.1 -c pytorch -c nvidia
pip install -r requirements.txt
```

where the CUDA version is set to `12.1`. For other CUDA versions, please refer to installation instructions of [PyTorch](https://pytorch.org/get-started/locally/). See [Trouble shooting](#trouble-shooting) for more details.

## Usage

Our implementation is based on HuggingFace `transformers`. We register a new model `lckv-llama` that supports the [Layer-Condensed KV Cache](https://arxiv.org/abs/2405.10637). It inherits from the `llama` model and adds support for the Layer-Condensed KV Cache.

```python
import models # register the lckv-llama model
from transformers import AutoModelForCausalLM, AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("huggyllama/llama-7b")
model = AutoModelForCausalLM.from_config(config="configs/tinyllama_lckv.json")
```

and now you have a randomly initialized model with the Layer-Condensed KV Cache.

### Optimization

To accelerate the training and inference of the model, one could apply the liger kernel supported by `transformers` library. The provided training script `run_clm.py` has already activated the liger kernel. See more details [here](https://huggingface.co/docs/transformers/v4.45.2/en/trainer#liger-kernel).

### Configuration

We provide some sample configuration files in the  `configs` folder. The config settings are defined in [models/configuration_lckv.py](models/configuration_lckv.py). You may refer to this file for more details.

#### Option 1: Modify the configurations in python:

```python
from models import LCKVLlamaConfig

# we have prepared a sample configuration file
config = LCKVLlamaConfig.from_pretrained("configs/tinyllama_lckv.json")

# below is the LCKV config. you may modify the configuration as you like
config.forward_passes  = 7      # m in the paper
config.backward_passes = 2      # b in the paper
config.layer_types     = "0_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_21" # for each layer, which layer to attend to

# we also support this
config.layer_types     = "0_10_10_10_10_10_10_10_10_10_10_10_10_10_10_10_10_10_10_10_10_21" # the sandwich-middle configuration
config.layer_types     = "0_1_2_3_4_5_6_7_8_9_10_11_12_13_14_15_16_17_18_19_20_21" # Llama config
config.layer_types     = "0_0_2_2_4_4_6_6_8_8_10_10_12_12_14_14_16_16_18_18_20_20" # CLA config

config.sliding_window  = 1024   # the window size for the sliding window attention
config.layer_types     = "0s_1s_2s_3s_4s_5s_6s_7s_8s_9s_10s_11_11_11_11_11_11_11_11_11_11_11" # YOCO config, 's' is for sliding window

config.sliding_window  = 1024   # the window size for the sliding window attention
config.layer_types     = "0_1s_1s_3s_3s_3s_0_7s_7s_9s_9s_9s_12_13s_13s_15s_15s_15s_12_19s_19s_19s" # MixAttention (Pairs) config

# we also support sequential training / inference, which will process the tokens one by one
# corresponding to LCKV paper Figure 2(a)
config.use_sequential = True
```

#### Option 2: Modify the configurations in the shell script (via `--config_overrides`):

```sh
accelerate launch run_clm.py \
    --config_name configs/tinyllama_lckv.json \
    --config_overrides forward_passes=7,backward_passes=2,layer_types=0_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_20_21 \
    ...
```

With the above configurations, you can create [CLA](http://arxiv.org/abs/2405.12981), [YOCO](https://arxiv.org/abs/2405.05254) or any configurations in [Cross-Layer KV Sharing](http://arxiv.org/abs/2410.14442) or [MixAttention](http://arxiv.org/abs/2409.15012) without changing the code. The only thing you need to do is to write the correct `layer_types` in the configuration file. You can also experiment on differnt types of configurations like our new custom configuration with inital few layers of CLA2 and last few layers as CLA3.

### Pre-training

We use the same [training script](https://github.com/huggingface/transformers/blob/main/examples/pytorch/language-modeling/run_clm.py) as the original `transformers` library. You may refer to the [official documentation](https://huggingface.co/transformers/training.html) for more details.

We provide a training script `run_clm.sh` for training a 50M parameter model on the `wikitext-103` dataset. You may run the script with:

```sh
bash run_clm.sh
```

See the script for more details. For pretraining on SlimPajama, please follow the instructions in [tinyllama-zh](https://github.com/whyNLP/tinyllama-zh) and replace the dataset with SlimPajama.


#### Initializing from a Pretrained Model

We may initialize our LCKV model from a pretrained model. Most parts of the model structure are consistent with the standard transformer model and we can directly inherit the weights. For the KV weights $W_K, W_V$, we mainly have 2 options:

##### Option 1: Directly Copy the Weights

Simply add `--model_name_or_path` to the training script:

```sh
accelerate launch run_clm.py \
    --model_name_or_path TinyLlama/TinyLlama-1.1B-intermediate-step-1195k-token-2.5T \
    --config configs/tinyllama_lckv.json \
    ...
```

See the script `run_clm.sh` for more details.

##### Option 2: Average the Weights from Multiple Layers

Following [MLKV](http://arxiv.org/abs/2406.09297), we may average the weights from multiple layers to initialize the KV weights. We provide a script `convert_pretrained.py` to convert the pretrained model to the LCKV model. You may run the following command:

```sh
python convert_pretrained.py --model_name_or_path TinyLlama/TinyLlama-1.1B-intermediate-step-1195k-token-2.5T --config_name configs/tinyllama_lckv.json --output_dir outputs/tinyllama-converted
```

The KV weights of each layer will be the average from the all the layers attends to it. For example,

```python
# the CLA / MLKV config
config.layer_types = "0_0_2_2_4_4_6_6"
# then layer 0 will have the average KV weights from layer 0 and 1 in the pretrained model
#      layer 2 will have the average KV weights from layer 2 and 3 in the pretrained model

# the LCKV config
config.layer_types = "0_6_6_6_6_6_6_7"
# then layer 0 will inherit the KV weights from layer 0 in the pretrained model
#      layer 6 will have the average KV weights from layer 1, 2, 3, 4, 5, 6 in the pretrained model
#      layer 7 will inherit the KV weights from layer 7 in the pretrained model
```

then, use the converted model to initialize the LCKV model:

```sh
accelerate launch run_clm.py \
    --model_name_or_path outputs/tinyllama-converted \
    ...
```

Our experiments show that such an initialization strategy can effectively improve the performance of the model in most cases.

## Code Style

We mostly follow that of `transformers`. Run the following command to check the code style:

```sh
# Use `pip install ruff` to install ruff if it is not available
ruff check models
```

See more details in `pyproject.toml`.


## Trouble shooting

### Flash-Attn Installation

https://github.com/Dao-AILab/flash-attention/issues/451

Behavior:

Runtime error.
```sh
ImportError: /home/.../flash_attn_2_cuda.cpython-38-x86_64-linux-gnu.so: undefined symbol: _ZN2at4_ops9_pad_enum4callERKNS_6TensorEN3c108ArrayRefINS5_6SymIntEEElNS5_...
```

Solution:
```sh
pip uninstall flash-attn
FLASH_ATTENTION_FORCE_BUILD=TRUE pip install flash-attn
```

### CUDA version

The cuda version may affect the installation of:
- [PyTorch](https://pytorch.org/get-started/locally/)
- [Flash-Attn](https://github.com/Dao-AILab/flash-attention)

Please make sure to install the correct version of the packages (so long as they are consistent, the code would work). Also make sure that `nvcc` is installed and available in the path.

Our experiment environment uses `CUDA 12.1` and you may install with
```sh
conda install pytorch==2.5.0 pytorch-cuda=12.1 -c pytorch -c nvidia
pip install -r requirements.txt
```

### Sequential update produces different outputs

Behavior: Model inference with sequential update will produce different outputs with parallel update.

This is due to the precision issues. We find that using `bfloat16`, the down projection in Llama MLP will produce different results when inference with different number of tokens.
