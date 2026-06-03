# vllman

`vllman` is a tiny lifecycle wrapper for running vLLM model servers in Podman containers from YAML files.

It is intentionally boring: one YAML file per model, one predictable container name per model, and a small set of commands that are easy to explain and remember.

```bash
vllman run gemma-4-31b
vllman logs -f gemma-4-31b
vllman stop gemma-4-31b
vllman remove gemma-4-31b
```

The goal is to make a cursed inference stack slightly less cursed.

## What it does

Given a model runner definition like:

```yaml
model: RedHatAI/gemma-4-31B-it-NVFP4
name: gemma-4-31b
runtime: vllm-base:260603-nightly-cu130

gpu: "0"

host: "127.0.0.1"
port: 8013

container_host: "::"
container_port: 8000

network: dualstack

options:
  tensor-parallel-size: 1
  trust-remote-code: true
  max-model-len: 131072
  gpu-memory-utilization: 0.47
  reasoning-parser: gemma4
  tool-call-parser: gemma4
  enable-auto-tool-choice: true
  kv-cache-dtype: fp8
  async-scheduling: true
  enable-chunked-prefill: true
  max-num-batched-tokens: 8192
  enable-prefix-caching: true
  limit-mm-per-prompt:
    image: 32
    audio: 0
  default-chat-template-kwargs:
    enable_thinking: false
  speculative-config:
    model: google/gemma-4-31B-it-assistant
    num_speculative_tokens: 4
```

`vllman run gemma-4-31b` starts a Podman container roughly equivalent to:

```bash
podman run -d \
  --name vllm-gemma-4-31b \
  --gpus 0 \
  --network dualstack \
  -v /root/.cache/huggingface:/root/.cache/huggingface \
  -p 127.0.0.1:8013:8000 \
  vllm-base:260603-nightly-cu130 \
  vllm serve RedHatAI/gemma-4-31B-it-NVFP4 \
    --host :: \
    --port 8000 \
    --tensor-parallel-size 1 \
    --trust-remote-code \
    --max-model-len 131072 \
    --gpu-memory-utilization 0.47 \
    --reasoning-parser gemma4 \
    --tool-call-parser gemma4 \
    --enable-auto-tool-choice \
    --kv-cache-dtype fp8 \
    --async-scheduling \
    --enable-chunked-prefill \
    --max-num-batched-tokens 8192 \
    --enable-prefix-caching \
    --limit-mm-per-prompt '{"image":32,"audio":0}' \
    --default-chat-template-kwargs '{"enable_thinking":false}' \
    --speculative-config '{"model":"google/gemma-4-31B-it-assistant","num_speculative_tokens":4}'
```

## Installation

Install the script somewhere on your `PATH`:

```bash
sudo install -m 0755 vllman /usr/local/bin/vllman
```

Create the model runner directory:

```bash
sudo mkdir -p /opt/modelrunners
```

Place model definitions there:

```bash
sudo cp gemma-4-31b.yaml /opt/modelrunners/gemma-4-31b.yaml
```

## Usage

### Run a model

Create and start the container:

```bash
vllman run gemma-4-31b
```

This reads:

```text
/opt/modelrunners/gemma-4-31b.yaml
```

and creates a container named:

```text
vllm-gemma-4-31b
```

### Test a model interactively

Run the container in the foreground with `--rm -it`:

```bash
vllman test gemma-4-31b
```

This is useful while tuning model arguments, checking startup logs, or debugging broken images.

### Start an existing container

```bash
vllman start gemma-4-31b
```

### Stop a container

```bash
vllman stop gemma-4-31b
```

### Remove a container

```bash
vllman remove gemma-4-31b
```

Alias:

```bash
vllman rm gemma-4-31b
```

This runs `podman rm -f`.

### View logs

```bash
vllman logs gemma-4-31b
```

Follow logs:

```bash
vllman logs -f gemma-4-31b
```

Show previous logs where supported by Podman:

```bash
vllman logs -p gemma-4-31b
```

### Show status

```bash
vllman status gemma-4-31b
```

### List available model definitions

```bash
vllman list
```

### Dry run

Print the Podman command without executing it:

```bash
vllman --dry-run run gemma-4-31b
vllman --dry-run test gemma-4-31b
```

This is strongly recommended when creating or changing a model YAML.

## YAML reference

Model definitions live in:

```text
/opt/modelrunners
```

The file name is the model runner name:

```text
/opt/modelrunners/gemma-4-31b.yaml
```

Run it with:

```bash
vllman run gemma-4-31b
```

### Required fields

#### `model`

The Hugging Face model ID or local model path passed to `vllm serve`.

```yaml
model: RedHatAI/gemma-4-31B-it-NVFP4
```

#### `runtime`

The container image to run.

```yaml
runtime: vllm-base:260603-nightly-cu130
```

#### `port`

The host port to expose.

```yaml
port: 8013
```

### Common fields

#### `name`

The logical model runner name.

```yaml
name: gemma-4-31b
```

If omitted, the YAML file name is used.

The container name is generated as:

```text
vllm-<name>
```

For example:

```text
vllm-gemma-4-31b
```

#### `gpu` / `gpus`

GPU selector passed to Podman.

```yaml
gpu: "0"
```

or:

```yaml
gpus: "0"
```

This emits:

```bash
--gpus 0
```

#### `host`

Host-side bind address for the published port.

For local access from LiteLLM or another service on the same machine, prefer IPv4 loopback:

```yaml
host: "127.0.0.1"
```

This emits:

```bash
-p 127.0.0.1:8013:8000
```

For wildcard IPv4:

```yaml
host: "0.0.0.0"
```

For IPv6:

```yaml
host: "::"
```

This emits:

```bash
-p '[::]:8013:8000'
```

Note: IPv6 host-port publishing through Podman bridge networking can be surprising, especially when curling from the same host through localhost. If local access is the only requirement, `127.0.0.1` is usually the least exciting choice.

#### `container_host`

The address vLLM binds to inside the container.

```yaml
container_host: "::"
```

This emits:

```bash
--host ::
```

#### `container_port`

The vLLM port inside the container.

```yaml
container_port: 8000
```

By convention, this should usually stay `8000`.

#### `network`

Podman network to use.

```yaml
network: dualstack
```

This emits:

```bash
--network dualstack
```

For host networking:

```yaml
network: host
```

When using host networking, Podman does not perform port mappings. In that case, set `container_port` to the actual port you want vLLM to listen on.

Example:

```yaml
network: host
container_host: "::"
container_port: 8013
```

#### `volumes`

Additional volume mounts.

```yaml
volumes:
  - /models:/models
  - /data/tokenizers:/data/tokenizers
```

By default, `vllman` mounts the Hugging Face cache:

```text
/root/.cache/huggingface:/root/.cache/huggingface
```

#### `env`

Environment variables to pass into the container.

```yaml
env:
  HF_TOKEN: hf_xxx
  VLLM_LOGGING_LEVEL: DEBUG
```

This emits:

```bash
-e HF_TOKEN=...
-e VLLM_LOGGING_LEVEL=DEBUG
```

#### `podman_args`

Additional raw arguments passed to `podman run`.

```yaml
podman_args:
  - --ipc=host
  - --shm-size=16g
```

Use this for Podman/container-level settings.

#### `options`

vLLM arguments passed to `vllm serve`.

```yaml
options:
  tensor-parallel-size: 1
  trust-remote-code: true
  max-model-len: 131072
```

YAML keys become CLI flags:

```text
tensor-parallel-size
```

becomes:

```bash
--tensor-parallel-size
```

Boolean `true` values are emitted as flags:

```yaml
trust-remote-code: true
```

becomes:

```bash
--trust-remote-code
```

Boolean `false` values are omitted.

Objects and arrays are serialized as compact JSON so that options like these work correctly:

```yaml
limit-mm-per-prompt:
  image: 32
  audio: 0

default-chat-template-kwargs:
  enable_thinking: false

speculative-config:
  model: google/gemma-4-31B-it-assistant
  num_speculative_tokens: 4
```

These become:

```bash
--limit-mm-per-prompt '{"image":32,"audio":0}'
--default-chat-template-kwargs '{"enable_thinking":false}'
--speculative-config '{"model":"google/gemma-4-31B-it-assistant","num_speculative_tokens":4}'
```

#### `extra_args`

Additional raw arguments passed to `vllm serve` after generated options.

```yaml
extra_args:
  - --some-vllm-flag
  - some-value
```

Use this for flags not yet represented in structured YAML.

## Example: Gemma 4 31B

```yaml
model: RedHatAI/gemma-4-31B-it-NVFP4
name: gemma-4-31b
runtime: vllm-base:260603-nightly-cu130

gpu: "0"

host: "127.0.0.1"
port: 8013

container_host: "::"
container_port: 8000

network: dualstack

options:
  tensor-parallel-size: 1
  trust-remote-code: true
  max-model-len: 131072
  gpu-memory-utilization: 0.47
  reasoning-parser: gemma4
  tool-call-parser: gemma4
  enable-auto-tool-choice: true
  kv-cache-dtype: fp8
  async-scheduling: true
  enable-chunked-prefill: true
  max-num-batched-tokens: 8192
  enable-prefix-caching: true
  limit-mm-per-prompt:
    image: 32
    audio: 0
  default-chat-template-kwargs:
    enable_thinking: false
  speculative-config:
    model: google/gemma-4-31B-it-assistant
    num_speculative_tokens: 4
```

After starting it:

```bash
vllman run gemma-4-31b
```

Test the OpenAI-compatible endpoint:

```bash
curl http://127.0.0.1:8013/v1/models
```

LiteLLM can then use:

```text
http://127.0.0.1:8013/v1
```

## Environment variables

### `VLLMAN_CONFIG_DIR`

Override the model runner config directory.

Default:

```text
/opt/modelrunners
```

Example:

```bash
VLLMAN_CONFIG_DIR=$PWD/modelrunners vllman list
```

### `VLLMAN_NETWORK`

Override the default Podman network.

Example:

```bash
VLLMAN_NETWORK=host vllman test gemma-4-31b
```

### `VLLMAN_CONTAINER_PREFIX`

Override the container name prefix.

Default:

```text
vllm-
```

Example:

```bash
VLLMAN_CONTAINER_PREFIX=model- vllman status gemma-4-31b
```

### `VLLMAN_CONTAINER_PORT`

Override the default in-container vLLM port.

Default:

```text
8000
```

Example:

```bash
VLLMAN_CONTAINER_PORT=9900 VLLMAN_NETWORK=host vllman test gemma-4-31b
```

## Networking notes

The recommended default for local host access is:

```yaml
host: "127.0.0.1"
port: 8013
container_host: "::"
container_port: 8000
network: dualstack
```

This means:

- vLLM listens on port `8000` inside the container
- Podman publishes it on `127.0.0.1:8013`
- local services such as LiteLLM can connect to `http://127.0.0.1:8013/v1`

IPv6 host-port publishing with Podman, Netavark, and bridge networking may not behave like IPv4 localhost publishing. If the container is reachable directly on its IPv6 container address but not through the published host port, try publishing on IPv4 loopback first:

```yaml
host: "127.0.0.1"
```

For host networking, remember that Podman does not apply `-p` mappings:

```yaml
network: host
container_port: 8013
```

## Troubleshooting

### Print the generated command

```bash
vllman --dry-run run gemma-4-31b
```

### Check whether vLLM is listening inside the container

```bash
podman exec -it vllm-gemma-4-31b ss -ltnp
```

### Test from inside the container

```bash
podman exec -it vllm-gemma-4-31b curl -v http://127.0.0.1:8000/v1/models
podman exec -it vllm-gemma-4-31b curl -g -v http://[::1]:8000/v1/models
```

### Test from the host

```bash
curl -v http://127.0.0.1:8013/v1/models
```

### Inspect Podman port publishing

```bash
podman ps
ss -ltnp | grep 8013
```

### Inspect the network

```bash
podman network inspect dualstack
podman info --format '{{json .Host.NetworkBackend}}'
podman info --format '{{json .Host.RootlessNetworkCmd}}'
```

### Inspect nftables rules

```bash
nft list ruleset | grep -E '8013|podman|netavark|cni'
```

## Design principles

`vllman` tries to stay small and predictable.

It does not try to be Kubernetes.

It does not try to replace Podman.

It does not try to understand every vLLM option.

It simply:

1. reads `/opt/modelrunners/<name>.yaml`
2. converts structured YAML into safe CLI arguments
3. starts or manages a predictable Podman container

For everything else, use `podman_args`, `extra_args`, or `--dry-run`.

## License

MIT License

Copyright (c) 2026 urbanstudio GmbH, Sven Kayser (sk@urbanstudio.de)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
