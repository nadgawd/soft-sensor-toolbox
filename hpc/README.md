# Running the LLM on the IITD HPC

Serves the assistant's model with vLLM on a GPU compute node, in place of the
cloud APIs. The backend reaches it in one of two ways:

- **Local development:** the backend runs on your laptop and talks to the
  compute node over an SSH tunnel.
- **The online app:** the laptop forwards the same tunnel to the internet
  through Tailscale Funnel, and the hosted backend calls that public URL with an
  API key.

Either way, the backend treats the HPC model as optional. If it is unreachable,
chat falls back to the cloud providers within a few seconds.

All settings live in one file, [`env.sh`](env.sh). Source it before anything
else; nothing else hardcodes a path.

```bash
source hpc/env.sh
```

Placeholders used below: `<kerberos-id>` is your cluster login,
`<project-code>` is the allocation you pass to `qsub -P`, and `<node>` is
whichever compute node PBS gives you.

## Configure `env.sh` first

The defaults in `env.sh` are one user's. Before running anything, set these to
your own, either by editing the file or by exporting them:

| Variable | Meaning |
|---|---|
| `HPC_USER` | Your Kerberos ID. Defaults to `whoami`. |
| `HPC_DEPT`, `HPC_PROGRAMME` | Your department and programme, which make up your scratch path. |
| `SCRATCH_DIR` | Or set the full scratch path directly. |
| `PROJECT_CODE` | Your `qsub -P` allocation. Ask your supervisor if you don't have one. |

The `#PBS -P` lines at the top of `serve_llm.pbs` and `train_lora.pbs` hold a
project code too, because PBS reads those directives before any script runs.
Change them, or pass `-P <project-code>` to `qsub`.

## Most of this cluster's GPUs cannot run vLLM

This is the single most important thing to know. A plain
`-l select=1:ncpus=8:ngpus=1` usually lands on a **Tesla K40m**: compute
capability 3.5, 11 GB, driver 470. vLLM requires 7.0 or higher, and modern
PyTorch is not even built for Kepler. No configuration makes it work, and these
cards are the majority of the GPU nodes.

PBS exposes no `gpu_model` resource here, but each CPU generation comes with
one GPU generation, so the `centos` resource works as a GPU selector:

| `centos` | GPU |
|---|---|
| `haswell` | Tesla K40m, **unusable** |
| `skylake` | V100-class |
| `icelake` | **A100-PCIE-40GB**, driver 590 |
| `amdepyc` | 8 GPUs per node, 1 TB RAM (restricted queue) |

`serve_llm.pbs` therefore requests `centos=icelake`. Keep the CPU chunk small.
Asking for 8 cores queued indefinitely behind *"Insufficient amount of
resource: ncpus"* while GPUs sat idle; 4 cores schedules quickly, and inference
is GPU-bound anyway.

## Three facts that explain most of the friction

**Scratch is not `/scratch/<kerberos-id>`.** It is divided by department and
programme: `/scratch/<dept>/<programme>/<kerberos-id>/mtp`. Scripts that guess
`/scratch/$(whoami)` produce a path you have no permission to create. Set
`HPC_DEPT` / `HPC_PROGRAMME` in `env.sh`, or export `SCRATCH_DIR` directly.

**The login nodes reach the internet through the campus proxy with no portal
login.** `env.sh` exports the proxy variables for you. Compute nodes are
stricter and need `proxy_login.sh`, but nothing in this workflow requires that,
because downloads happen on the login node. Without the proxy,
huggingface_hub hangs silently rather than failing, which is why `env.sh`
enables it by default.

**Serving needs no internet at all.** The weights sit on scratch, and
`HF_HUB_OFFLINE=1` stops vLLM from calling the Hub. So the serve job is an
ordinary batch `qsub`, with no browser login to keep alive.

## One-time setup

Run on a **login node**. It needs no GPU and downloads about 4 GB of wheels.

```bash
cd ~/mtp
source hpc/env.sh
bash hpc/setup_scratch.sh    # conda env on scratch (~20 min, ~10 GB)
bash hpc/fetch_model.sh      # Qwen3.5-27B-GPTQ-Int4, 29 GB
MODEL_PROFILE=qwen2.5-7b bash hpc/fetch_model.sh   # optional 7B fallback, 16 GB
bash hpc/make_llm_key.sh     # API key for the server; never printed
```

Run the long steps under `tmux`, so a dropped SSH session doesn't leave a
half-built environment behind.

The env is created with `conda create --prefix` on scratch, not `--name`: a
named env goes to `~/.conda/envs`, and torch alone would exhaust the home
quota. `CONDA_PKGS_DIRS` and `PIP_CACHE_DIR` are redirected for the same
reason.

Scripts call `$ENV_PY`, an absolute interpreter path, rather than activating the
env. In a non-interactive shell, `conda activate` can return success while
leaving `PATH` untouched. `pip` then silently resolves to the read-only system
conda and dies with `EACCES`.

### Two path traps worth knowing

The `apps/miniconda` module exports
`PYTHONPATH=<base>/lib/python3.12/site-packages`. That leaks into *every*
interpreter afterwards, including the env's Python 3.10, so pip finds base
packages, tries to uninstall root-owned files, and fails with `EACCES`. It is
also an ABI mismatch. `load_conda` in `env.sh` unsets it.

Old `pip install --user` packages in `~/.local` similarly shadow the env;
`PYTHONNOUSERSITE=1` disables that path. To reclaim the quota:
`rm -rf ~/.local/lib/python3.*/site-packages`.

## SSH: sign in once, reuse the connection

Every script drives the cluster through one shared SSH connection. Add this to
your laptop's `~/.ssh/config`:

```
Host iitd
    HostName hpc.iitd.ac.in
    User <kerberos-id>
    ControlMaster auto
    ControlPath ~/.ssh/cm/%r@%h:%p
    ControlPersist 8h
```

Create the socket directory once with `mkdir -p ~/.ssh/cm`, then sign in
interactively with `ssh iitd`. Every later `ssh`, `rsync` and tunnel reuses that
connection without prompting. If you close that first window, the socket goes
away and everything prompts for a password again.

Commands sent over this connection run in a non-interactive shell, where
`qstat` and `qsub` aren't on `PATH`. Wrap them in a login shell:

```bash
ssh iitd 'bash -lc "qstat -u \$USER"'
```

## Local development

From the project root, one command does all of it:

```bash
./run.sh              # GPU job + tunnel + backend + frontend
./run.sh --stop       # tear down local processes (the PBS job survives)
./run.sh --no-cluster # laptop only; chat uses the cloud keys in .env
```

It starts the model profile named by `LOCAL_LLM_EXECUTION_MODEL` in `.env`,
reusing a running or queued job for that profile (`vllm_qwen3527b` or
`vllm_qwen257b`) rather than submitting a second one. It warns, with a `qdel`
command, about LLM jobs of yours for any other profile that are still holding a
GPU. It reads the compute node from PBS rather than the buffered job log, and
waits for the model to finish loading before it starts the backend. Ctrl-C
stops the local processes and leaves the GPU job running, since restarting one
costs several minutes.

Point the backend at it in `.env`:

```
LLM_MODE=local_first
LOCAL_LLM_BASE_URL=http://localhost:8888/v1
LOCAL_LLM_API_KEY=<contents of hpc/.llm_api_key>
LOCAL_LLM_EXECUTION_MODEL=qwen3.5-27b
LOCAL_LLM_KNOWLEDGE_MODEL=qwen3.5-27b
LOCAL_LLM_ROUTER_MODEL=qwen3.5-27b
LOCAL_LLM_THINKING=1
```

To fall back to the 7B, set the three model lines to `qwen2.5-7b`, remove
`LOCAL_LLM_THINKING`, and re-run `./run.sh`; it starts the matching job.

`local_first` is the right mode while the cluster is the weak link. The tunnel
dies when the walltime expires, and this mode falls back to a cloud key rather
than breaking chat mid-session.

<details>
<summary>The same thing by hand</summary>

```bash
# 1. on the cluster (default profile, qwen3.5-27b)
cd ~/mtp && qsub hpc/serve_llm.pbs
qstat -u "$USER"
grep "Running on node" hpc/logs/vllm_qwen3527b.out   # -> <node>

# 2. on your laptop
COMPUTE_NODE=<node> bash hpc/tunnel.sh               # leave open

# 3. another laptop terminal
bash hpc/check_server.sh localhost 8888
source .venv_run/bin/activate && uvicorn backend.main:app --reload --port 8000
cd frontend && npm run dev
```
</details>

One subtlety if you tunnel by hand: when a ControlMaster socket exists, a `-L`
forward belongs to the *master* process, not the client that asked for it.
Killing the client leaves the port bound, and your next tunnel dies silently.
Release it with `ssh -O cancel -L 8888:<node>:8000 iitd`.

`check_server.sh` does more than ping `/v1/models`: it sends a real tool-call
request. A server that chats perfectly can still fail to emit tool calls when
`--tool-call-parser` doesn't match the model. That failure is invisible until
the agent silently stops calling tools.

## Serving the online app

```bash
./run.sh --serve-llm
```

This runs only the cluster side; the backend and frontend are hosted
elsewhere (see [`DEPLOY.md`](../DEPLOY.md)). In order, it:

1. Submits or reuses the serve job, then waits up to 10 minutes for PBS to start
   it.
2. Forwards `localhost:8888` to the node, then waits up to 15 minutes for the
   model to load.
3. Refuses to continue if the server answers without the API key, because the
   next step makes it public.
4. Publishes `localhost:8888/v1` at `https://<mac>.<tailnet>.ts.net/v1` with
   Tailscale Funnel.
5. Keeps the Mac awake (`caffeinate`) while it runs.
6. Watches the job. When the walltime ends, it submits a new job and re-forwards
   the port. If the tunnel stops answering, it forwards the port again.

The hosted backend needs two settings: `LOCAL_LLM_BASE_URL` set to the public
URL the script prints, and `LOCAL_LLM_API_KEY` set to the same key as
`hpc/.llm_api_key`. If the key doesn't match, the HPC rejects the backend's
calls, and the app quietly uses the cloud models even though the server is up.

Ctrl-C turns the Funnel off and leaves the PBS job running. While the Funnel is
off, the Mac is asleep, or a new job is loading, the online app answers from the
cloud providers. The backend retries the HPC every 30 seconds and switches back
by itself once it answers.

**Walltime.** `serve_llm.pbs` asks for 4 hours. The standard queue allows up to
168; longer requests queue slightly longer and use more of your allocation. The
watchdog only helps while the Mac stays on and signed in to `ssh iitd`. If that
SSH connection drops, it prints a warning and waits for you to sign in again.

**A busy queue.** If the A100 nodes are full, the script stops with
`job ... still not running` after 10 minutes. The job stays queued, so run
`./run.sh --serve-llm` again later; it picks up the same job instead of
submitting another.

## Which model

`env.sh` defines two profiles. The profile name is also the name the model is
served under, which is what `.env` refers to.

| | `qwen3.5-27b` (default) | `qwen2.5-7b` (fallback) |
|---|---|---|
| Checkpoint | `Qwen/Qwen3.5-27B-GPTQ-Int4` | `Qwen/Qwen2.5-7B-Instruct` |
| GPU memory for weights | 26.7 GiB | ~15 GiB |
| Context | 64k (KV cache holds ~137k tokens) | 32k |
| Decode speed, one stream | ~42 tok/s compiled | fast either way |
| Tool-call parser | `qwen3_coder` | `hermes` |
| Thinking mode | yes, toggled per request | no |

The 7B was the bring-up model: unquantized, quick to download, and easy to debug
the firewall, PBS, tunnel and tool parsing against. The 27B is much better at
the agent's actual work: choosing the right tool from about 25, filling
arguments with real column names, and chaining several calls.

The 27B's flags are explained at the bottom of `serve_llm.pbs`. Two are easy to
get wrong. It needs `--quantization gptq_marlin`, not the model card's
`moe_wna16`. `moe_wna16` only quantizes MoE experts, so it loads this dense
model's MLPs in bf16 and runs out of memory. And it compiles by default:
eager mode decodes at about 12 tok/s, against about 42 compiled.

**Startup** takes several minutes. Most of it is reading the weights from
scratch, which is slow on a node that hasn't read them recently. Then comes
torch.compile (about 50 s the first time, cached on scratch afterwards),
profiling, and KV-cache setup. The server is silent until it is ready, and PBS
buffers job output, so poll `curl http://<node>:8000/v1/models` from the login
node rather than watching the log.

**Thinking.** Qwen3.5 can reason before answering. The server starts with
thinking off, and the chat's "Think" toggle turns it on for one request. The
backend then sends `enable_thinking: true` with the model card's sampling
settings. The reasoning streams into the activity log, separate from the
answer. It makes replies noticeably slower, so keep it for hard questions.

**Other GPUs.** The compute capability, printed at the top of every serve log,
decides what works. The GPTQ Marlin kernels the 27B uses need 8.0 or higher
(A100 and newer). On a 7.0 V100, use the 7B profile.

Adding a profile means a new entry in the `case` blocks of `env.sh` and
`serve_llm.pbs`, then `MODEL_PROFILE=<name> bash hpc/fetch_model.sh`.

## Security

- The server requires an API key whenever `hpc/.llm_api_key` exists.
  `make_llm_key.sh` creates it with mode 600 and never prints it.
  `serve_llm.pbs` passes it to vLLM through an environment variable, so it
  doesn't show up in `ps` on the shared node.
- `run.sh` does not copy the key file when it syncs `hpc/` to the cluster, and
  the file is gitignored. To rotate the key, delete the file, run
  `make_llm_key.sh`, and restart the job. Then update `LOCAL_LLM_API_KEY` in
  `.env` and on the hosted backend.
- `proxy_login.sh` reads your Kerberos password with a hidden prompt. Never pass
  it on a command line, where it would end up in your shell history and in `ps`.

## On fine-tuning

Probably unnecessary. The router settles most intents in a regex layer that
never calls a model, with about forty phrasings pinned in
`tests/test_router.py`, including cases where a theory question reuses an
action verb. Only genuinely ambiguous turns reach the model, and the served
model handles those.

`train_lora.pbs` is there for when logged conversations show the served model
mishandling real tool arguments; use those turns as the training set. The
format is in `data/soft_sensor_train.jsonl.example`. You cannot LoRA-train a
quantized checkpoint, so train against the unquantized base with `--use_4bit`,
merge with `merge_lora.py`, and serve the merged result.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `cd: /scratch/<kerberos-id>: No such file` | Missing department/programme prefix; see `SCRATCH_DIR` in `env.sh` |
| `qstat: command not found` over SSH | Non-interactive shell; use `ssh iitd 'bash -lc "qstat -u \$USER"'` |
| `job ... still not running` from `run.sh` | Queue is busy; the job stays queued, so re-run later |
| Job stuck `Insufficient amount of resource: ncpus` | CPU chunk too big for the busy A100 nodes; 4 is enough |
| Job queued on `Insufficient amount of resource: ngpus` | No free A100; an old LLM job of yours may hold one (`run.sh` lists them) |
| vLLM: `no kernel image is available` | You landed on a K40m; the request lost `centos=icelake` |
| `PermissionError: 'ninja'` / `'nvcc'` | Env `bin/` not on `PATH`; `serve_llm.pbs` adds it |
| `CUDA compiler and CUDA toolkit headers are incompatible` | flashinfer JIT; `VLLM_USE_FLASHINFER_SAMPLER=0` |
| pip `EACCES` on the system miniconda | The module's `PYTHONPATH` leak; `load_conda` unsets it |
| Download hangs with no output | Proxy not exported; re-source `env.sh` |
| pip `302 Moved Temporarily` | Compute node not authenticated; `bash hpc/proxy_login.sh` |
| Model loads, never calls tools | Wrong `--tool-call-parser`; Qwen3.5 needs `qwen3_coder`, Qwen2.5 `hermes` |
| `<think>` text shows up in answers | `--reasoning-parser qwen3` missing |
| 404 on every chat turn | `LOCAL_LLM_*_MODEL` doesn't match the served profile name |
| Online app uses the cloud while the HPC is up | Hosted backend's `LOCAL_LLM_API_KEY` differs from `hpc/.llm_api_key` |
| `run.sh` refuses: job answers without the key | The job started before the key existed; `qdel` it and re-run |
| Tunnel open, connection refused | Job ended; `qstat -u $USER`, re-submit, and note the new node name |
| 27B OOM while creating weights | `--quantization moe_wna16` instead of `gptq_marlin` |
| CUDA OOM at load | Add `-v MAX_MODEL_LEN=32768` to the `qsub`, then lower `--gpu-memory-utilization` |

Scratch is not backed up and is purged after a period of inactivity. Model
weights are cheap to download again, but anything you care about belongs in
`$HOME` or off the cluster. Confirm the exact purge window with
`hpchelp@iitd.ac.in`.
