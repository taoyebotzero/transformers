<!--Copyright 2024 The HuggingFace Team. All rights reserved.
Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at
http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.
-->

# TorchAO

[TorchAO](https://github.com/pytorch/ao) is an architecture optimization library for PyTorch, it provides high performance dtypes, optimization techniques and kernels for inference and training, featuring composability with native PyTorch features like `torch.compile`, FSDP etc.. Some benchmark numbers can be found [here](https://github.com/pytorch/ao/tree/main/torchao/quantization#benchmarks).

Before you begin, make sure the following libraries are installed with their latest version:

```bash
# Updating 🤗 Transformers to the latest version, as the example script below uses the new auto compilation
pip install --upgrade torch torchao transformers
```

By default, the weights are loaded in full precision (torch.float32) regardless of the actual data type the weights are stored in such as torch.float16. Set `torch_dtype="auto"` to load the weights in the data type defined in a model's `config.json` file to automatically load the most memory-optimal data type.


## Manually Choose Quantization Types and Settings

`torchao` Provides many commonly used types of quantization, including different dtypes like int4, float8 and different flavors like weight only, dynamic quantization etc., only `int4_weight_only`, `int8_weight_only` and `int8_dynamic_activation_int8_weight` are integrated into hugigngface transformers currently, but we can add more when needed.
If you want to run the following codes on CPU even with GPU available, just change `device_map="cpu"` and `quantization_config = TorchAoConfig("int4_weight_only", group_size=128, layout=Int4CPULayout())` where `layout` comes from `from torchao.dtypes import Int4CPULayout` which is only available from torchao 0.8.0 and higher.

Users can manually specify the quantization types and settings they want to use:

```py
import torch
from transformers import TorchAoConfig, AutoModelForCausalLM, AutoTokenizer

model_name = "meta-llama/Meta-Llama-3-8B"
# We support int4_weight_only, int8_weight_only and int8_dynamic_activation_int8_weight
# More examples and documentations for arguments can be found in https://github.com/pytorch/ao/tree/main/torchao/quantization#other-available-quantization-techniques
quantization_config = TorchAoConfig("int4_weight_only", group_size=128)
quantized_model = AutoModelForCausalLM.from_pretrained(model_name, torch_dtype="auto", device_map="auto", quantization_config=quantization_config)

tokenizer = AutoTokenizer.from_pretrained(model_name)
input_text = "What are we having for dinner?"
input_ids = tokenizer(input_text, return_tensors="pt").to(quantized_model.device)

# auto-compile the quantized model with `cache_implementation="static"` to get speedup
output = quantized_model.generate(**input_ids, max_new_tokens=10, cache_implementation="static")
print(tokenizer.decode(output[0], skip_special_tokens=True))

# benchmark the performance
from torch._inductor.utils import do_bench_using_profiling
from typing import Callable

def benchmark_fn(func: Callable, *args, **kwargs) -> float:
    """Thin wrapper around do_bench_using_profiling"""
    no_args = lambda: func(*args, **kwargs)
    time = do_bench_using_profiling(no_args)
    return time * 1e3

MAX_NEW_TOKENS = 1000
print("int4wo-128 model:", benchmark_fn(quantized_model.generate, **input_ids, max_new_tokens=MAX_NEW_TOKENS, cache_implementation="static"))

bf16_model = AutoModelForCausalLM.from_pretrained(model_name, device_map="auto", torch_dtype=torch.bfloat16)
output = bf16_model.generate(**input_ids, max_new_tokens=10, cache_implementation="static") # auto-compile
print("bf16 model:", benchmark_fn(bf16_model.generate, **input_ids, max_new_tokens=MAX_NEW_TOKENS, cache_implementation="static"))

```

## Automatically Select Quantization Types

`torchao` also provies `autoquant` feature that automatically chooses a quantization type for quantizable layers such as linear based on microbenchmarks of quantizing and compiling a single linear layer.

```py
import torch
from transformers import TorchAoConfig, AutoModelForCausalLM, AutoTokenizer

model_name = "meta-llama/Meta-Llama-3-8B"
quantization_config = TorchAoConfig("autoquant", min_sqnr=None)
quantized_model = AutoModelForCausalLM.from_pretrained(model_name, torch_dtype="auto", device_map="auto", quantization_config=quantization_config)

tokenizer = AutoTokenizer.from_pretrained(model_name)
input_text = "What are we having for dinner?"
input_ids = tokenizer(input_text, return_tensors="pt").to("cuda")

# auto-compile the quantized model with `cache_implementation="static"` to get speedup
output = quantized_model.generate(**input_ids, max_new_tokens=10, cache_implementation="static")
# Due to some implementation details we are explicitly calling this now, we may refactor our code and remove this in the future
quantized_model.finalize_autoquant()
print(tokenizer.decode(output[0], skip_special_tokens=True))

# benchmark the performance
from torch._inductor.utils import do_bench_using_profiling
from typing import Callable

def benchmark_fn(func: Callable, *args, **kwargs) -> float:
    """Thin wrapper around do_bench_using_profiling"""
    no_args = lambda: func(*args, **kwargs)
    time = do_bench_using_profiling(no_args)
    return time * 1e3

MAX_NEW_TOKENS = 1000
print("autoquantized model:", benchmark_fn(quantized_model.generate, **input_ids, max_new_tokens=MAX_NEW_TOKENS, cache_implementation="static"))

bf16_model = AutoModelForCausalLM.from_pretrained(model_name, device_map="cuda", torch_dtype=torch.bfloat16)
output = bf16_model.generate(**input_ids, max_new_tokens=10, cache_implementation="static") # auto-compile
print("bf16 model:", benchmark_fn(bf16_model.generate, **input_ids, max_new_tokens=MAX_NEW_TOKENS, cache_implementation="static"))

```

## Serialization and Deserialization
torchao quantization is implemented with [tensor subclasses](https://pytorch.org/docs/stable/notes/extending.html#subclassing-torch-tensor), it only work with huggingface non-safetensor serialization and deserialization. It relies on `torch.load(..., weights_only=True)` to avoid arbitrary user code execution during load time and use [add_safe_globals](https://pytorch.org/docs/stable/notes/serialization.html#torch.serialization.add_safe_globals) to allowlist some known user functions.

The reason why it does not support safe tensor serialization is that wrapper tensor subclass allows maximum flexibility so we want to make sure the effort of supporting new format of quantized Tensor is low, while safe tensor optimizes for maximum safety (no user code execution), it also means we have to make sure to manually support new quantization format.

```py
# save quantized model locally
output_dir = "llama3-8b-int4wo-128"
quantized_model.save_pretrained(output_dir, safe_serialization=False)

# push to huggingface hub
# save_to = "{user_id}/llama3-8b-int4wo-128"
# quantized_model.push_to_hub(save_to, safe_serialization=False)

# load quantized model
ckpt_id = "llama3-8b-int4wo-128"  # or huggingface hub model id
loaded_quantized_model = AutoModelForCausalLM.from_pretrained(ckpt_id, device_map="auto")


# confirm the speedup
loaded_quantized_model = torch.compile(loaded_quantized_model, mode="max-autotune")
print("loaded int4wo-128 model:", benchmark_fn(loaded_quantized_model.generate, **input_ids, max_new_tokens=MAX_NEW_TOKENS))
```
