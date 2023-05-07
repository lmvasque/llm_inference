## LLM Inference

Experimental repository for inference with LLMs supported by HuggingFace.

## Setup

To set up the environment, run the following commands:

```bash
# create a directory for source code
mkdir installs

# create a directory for data, models, outputs, etc.
mkdir -p resources/data resources/outputs resources/models
# or alternatively, create a symlink
ln -s <path_to_storage> resources

# if working on slurm cluster, load relevant modules
ml multigpu anaconda3

# create a clean conda environment
# NOTE: we recommend using CUDA 11.6, but others may work too
conda create -n llm_hf1 -c conda-forge python=3.9 cudatoolkit=11.6 cudatoolkit-dev=11.6 -y
conda activate llm_hf1

# install transformers from source
git clone https://github.com/huggingface/transformers.git installs/transformers
cd installs/transformers
pip install -e .
cd ../..

# NOTE: for efficient inference, we use 8bit quantization with bitsandbytes. 
# This requires Turing or Ampere GPUs (RTX 20s, RTX 30s, A40-A100, T4+)
# install bitsandbytes from source
git clone https://github.com/TimDettmers/bitsandbytes.git installs/bitsandbytes
cd installs/bitsandbytes
CUDA_VERSION=116 make cuda11x
python setup.py install
cd ../..

# install other deps
pip install -r requirements.txt

# check the install and CUDA dependencies
python -m bitsandbytes

# For evaluation purposes, we also require the following packages
git clone https://github.com/feralvam/easse.git installs/easse
cd installs/easse
pip install -e .
cd ../..

git clone https://github.com/Yao-Dou/LENS.git installs/LENS
cd LENS/lens
pip install -e .
cd ../../..

```


## Data

To get the relevant datasets for reproducing experiments, use the script `scripts/fetch_datasets.sh`. 
This script downloads the raw data for publicly available datasets and writes the files to the `resources/data/` directory.

```bash
bash scripts/fetch_datasets.sh
```

Once you have downloaded the raw datasets, we can prepare them for inference using the relevant `prepare_*.py` script.
For example, to prepare ASSET, run

```bash
python -m scripts.prepare_asset
```

See the [readme](./scripts/README.md) for more details on the format.

## Examples

#### Hugging Face Models

To run inference with LLMs available on HuggingFace, run `inference.py` passing the relevant arguments, e.g.:

```bash
python -m inference \
	--model_name_or_path "bigscience/bloom-1b1" \
	--max_new_tokens 100 \
	--max_memory 0.65 \
	--batch_size 8 \
	--num_beams 1 \
	--num_return_sequences 1 \
	--do_sample True \
	--top_p 0.9 \
	--input_file "resources/data/asset/dataset/asset.test.jsonl" \
	--examples "resources/data/asset/dataset/asset.valid.jsonl" \
	--n_refs 1 \
	--few_shot_n 3 \
	--output_dir "resources/outputs" \
	--prompt_json "prompts/p0.json"
```

where:
- `--input_file` is a either a .txt file, with one input sentence per line or a JSONL file produced by `scripts.prepare_*.py`. For consistency we recommend the latter.
- `--examples` is a JSONL file produced by `scripts.prepare_*.py`, containing validation set examples that may be selected as few-shot examples.
- `--prompt_prefix` is a string prefix used by LangChain to construct the prompt.
- NB I: by default, an additional JSON file will be generated which persists the inference parameters used for generation.
- NB II: specify `--output_dir ''` to print your outputs to stdout (good for debugging/development purposes)


<!-- #### LLaMA

We can also use the same script to run inference with [LLaMA](https://github.com/facebookresearch/llama). If you have access to LLaMA, simply set the `--model_name_or_path` to the location of your local copy of the LLaMA weights and run the script with `torchrun`, e.g.:

```bash
python -m torch.distributed.run \
	--nproc_per_node 1 inference.py \
	--model_name_or_path <path-to-llama-model-7B> \
	--max_new_tokens 100 \
	--batch_size 8 \
	--top_p 0.9 \
	--examples "resourcesdata/asset/dataset/asset.valid.jsonl" \
	--input_file "resourcesdata/asset/dataset/asset.test.head10.jsonl" \
	--n_refs 1 \
	--few_shot_n 3 \
	--output_dir "resources/outputs" \
	--prompt_json "prompts/p0.json"
```

- NB: `--nproc_per_node` must equal the number of model shards (7B=1, 13B=2, 30B=4, 66B=8) -->


#### Running Models behind APIs

The script `inference.py` also supports running OpenAI or Cohere models.
You will have to modify the template of `secrets.py` such that `COHERE_API_KEY` and `OPENAI_API_KEY` are exposed to the library.

Running models can be done, for example, with the following command:
```bash
python -m inference \
--model_name_or_path cohere-command-xlarge-nightly \
--input_file "resources/data/asset/dataset/asset.test.jsonl" \
--examples "resources/data/asset/dataset/asset.valid.jsonl" \
--n_refs 1 \
--few_shot_n 3 \
--output_dir "resources/outputs" \
--prompt_json "prompts/p0.json"
```

Be aware that API models cost money! The approximate cost of running the OpenAI models on the ASSET test set (using 3 Few-shot examples with the pre-defined prompts) is

|          Model             |  `p0`  |  `p1`  |  `p2`  |        Pricing        |
| -------------------------- | ------ | ------ | ------ | --------------------- |
| openai-gpt-3.5-turbo       | $0.172 | $0.189 | $0.22  | $0.002 / 1k tokens    |
| openai-text-ada-001        | $0.035 | $0.038 | $0.046 | $0.0004 / 1k tokens   |
| openai-text-babbage-001    | $0.044 | $0.047 | $0.057 | $0.0005 / 1k tokens   |
| openai-text-curie-001      | $0.175 | $0.19  | $0.23  | $0.002 / 1k tokens    |
| openai-text-davinci-002    | $1.75  | $1.85  | $2.26  | $0.02 / 1k tokens     |
| openai-text-davinci-003    | $1.75  | $1.85  | $2.26  | $0.02 / 1k tokens     |
| approx. # TOKENS processed |  ~86K  |  ~93K  |  ~113k |                       |

## Prompting

To construct prompts flexibly, we use [LangChain](https://github.com/hwchase17/langchain).

A valid prompt may look something like the following:

```
I want you to replace my complex sentence with simple sentence(s). Keep the meaning same, but make them simpler.

Complex: The Hubble Space Telescope observed Fortuna in 1993.
Simple: 0: The Hubble Space Telescope spotted Fortuna in 1993.

Complex: Order # 56 / CMLN of 20 October 1973 prescribed the coat of arms of the Republic of Mali.
Simple: 0: In 1973, order #56/CMLN described the coat of arms for the Republic of Mali.

Complex: One side of the armed conflicts is composed mainly of the Sudanese military and the Janjaweed, a Sudanese militia group recruited mostly from the Afro-Arab Abbala tribes of the northern Rizeigat region in Sudan.
Simple:
```

This example corresponds to the T1 prompt described in [Feng et al., 2023](http://arxiv.org/abs/2302.11957).

Prompts can be defined on-the-fly at inference time by passing the relevant arguments. To do this for the example prompt above, pass the following arguments:

```bash
--prompt_prefix "I want you to replace my complex sentence with simple sentence(s). Keep the meaning same, but make them simpler."
--promt_suffix "Complex: {input}\nSimple:"
--prompt_template "Complex: {complex}\nSimple: {simple}"
--example_separator "\n\n"
--prompt_format "prefix_initial"
```

However, for reproducibility, we recommend using pre-defined prompts. These contain these relevant fields and easily be used for inference by passing them with the `--prompt_json` argument.
The directory [prompts](./prompts) contains a set of pre-defined prompts in JSON format. See the [readme](./prompts/README.md) for more details.

#### Experiments

For experiments, we provide a wrapper script, `run.py` that executes an inference run followed by evaluation of model outputs. For example, to run inference on ASSET with `bloom-560m`, you can run:

```bash
python -m run \
    --use_slurm True \
    --ntasks 1 \
    --cpus_per_task 1 \
    --gres gpu:T4:1 \
    --mem 20GB \
    --time 00:30:00 \
    --batch_size 8 \
    --seed 489 \
    --model_name_or_path bigscience/bloom-560m \
    --examples resources/data/asset/dataset/asset.valid.jsonl \
    --input_file resources/data/asset/dataset/asset.test.jsonl \
    --prompt_json prompts/p0.json \
	--n_refs 1 --few_shot_n 3
```

Alternatively, you can also pass a json file in position 1 with some or all of the arguments predefined. For example, on a server with RTX 3090 (24GB) GPUs, you could use the following:

```bash
python -m run exp_configs/rtx/bloom-560m.json \
    --seed 489 \
    --examples resources/data/asset/dataset/asset.valid.jsonl \
    --input_file resources/data/asset/dataset/asset.test.jsonl \
    --prompt_json prompts/p0.json \
	--n_refs 1 --few_shot_n 3
```

This script will produce the following files to help track experiments:

    - `<output_file>.jsonl`: The predictions of the model on the input file.
    - `<output_file>.json`: Command line arguments used for the inference run.
    - `<output_file>.log`: Log file of the inference run.
    - `<output_file>.eval`: Log file of the automatic evaluation with results. 


## Results

We have run a list of experiments to test simplification models on current models and datasets. The experiments' results can be seen in a [summarised format](https://github.com/tannonk/llm_simplification_results/tree/main/reports/summary) and [full format](https://github.com/tannonk/llm_simplification_results/tree/main/reports/full).
Please contact Tannon Kew (Slack) to get access to this repo.

## Limitations & Known Issues

- LLMs don't know when to stop. Thus, they typically generate sequences up to the specified `max_new_tokens`. The function `postprocess_model_outputs()` is used to extract the single relevant model output from a long generation sequence and is currently pretty rough.
- Setting `--n_refs` > 1 allows for a few-shot prompt example to have multiple possible targets (e.g. sampled from multiple validation set reference sentences). The current method of handling these is to enumerate them starting at 0, but this doesn't seem very elegant or intuitive.

## TODOs

You can find a list of pending experiments in this [checklist](https://github.com/tannonk/llm_inference/blob/main/reports/checklist/checklist.md). Feel free to suggest any new setting, model or dataset.