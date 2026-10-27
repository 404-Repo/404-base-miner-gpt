# 404-base-miner

Generate Three.js assets from images using your witness Pod and an OpenAI
subscription or API key. Python runs locally; the Pod handles model calls and
rendering. The selected JavaScript is submitted to the Pod.

Each image gets three candidates, a selection, two independent refinements and a
final selection. A successful run uses seven model calls and five previews.
Each candidate renders as soon as its model call finishes. Candidates run
concurrently; selection waits for the previews it needs.
Failed candidates are discarded while usable candidates continue. Failed selection
keeps the first valid candidate or the initial winner. A single candidate needs no
selection call. If every initial candidate fails, the prompt remains pending for
another runner pass.

## Setup

Use Python 3.13 and Poetry 1.8 or newer. Dependencies are pinned in `pyproject.toml`.

The witness and renderer SDK packages are not published on PyPI. They are
installed from the matching `404-subnet` source at revision
`ddca40dbb474900630798ae9492f89e4a69e59e9`. Copy `witness-client/`,
`renderer-client/` and `witness-server/verifier/params/` from that revision into
the sibling `../404-sdk/` directory. The Pod needs matching code and its round's
frozen mapping from image hashes to official prompt IDs.

Build the witness client's native library with Go 1.27.3 or newer and a C compiler
before installing. From this project directory, on macOS:

```sh
(cd ../404-sdk/witness-client/native && \
  go build -trimpath -buildmode=c-shared -o ../src/witness_client/libwitnessproof.dylib .)
poetry env use python3.13
poetry install
```

On Linux, use `libwitnessproof.so` instead of `libwitnessproof.dylib`.

Copy the matching public proof parameters into the ignored `proof-parameters/`
directory, or set their existing directory in `witness.json`:

```sh
mkdir -p proof-parameters
cp ../404-sdk/witness-server/verifier/params/{circuit,proving,verifying}.bin proof-parameters/
```

Use a registered Finney netuid 17 sr25519 hotkey in your Bittensor wallet.
Configure the gateway and local paths in the project-root `witness.json`.
Use `codex_home` for another Codex login directory.
Authentication defaults to your existing Codex ChatGPT login.
Set `OPENAI_API_KEY` and add `--api` for API billing.

## Generate

Pass the published round seed via `--seed` for either input mode. These examples
use [round 17's seed](https://github.com/404-Repo/404-active-competition/blob/main/rounds/17/seed.json):

```sh
poetry run python main.py --image /path/to/PROMPT_ID.png --seed 1313431840 \
  --wallet-name miner --wallet-hotkey default
```

`--image` accepts a file or HTTP(S) URL and can be repeated. Prompt IDs come from
filenames without the extension; keep the official filenames when downloading images.
Completed IDs are skipped before images are loaded. Generation receives the original bytes.

The first candidate uses `gpt-6-astra`; all other calls use `gpt-6.1-sol`.
Output token limits are 32768 for generation and 2048 for selection. Reasoning
is medium. Your account and the round must permit the chosen models.

For a batch, download the round's
[`prompts.txt`](https://github.com/404-Repo/404-active-competition/blob/main/rounds/17/prompts.txt).
It contains one image URL per line.

```sh
poetry run python main.py --prompts /path/to/prompts.txt --seed 1313431840 \
  --parallel-prompts 8 --parallel-calls 12
```

The default is 8 prompts. The SDK limits active model calls according to
`--parallel-calls` and handles preparation concurrency. Render requests wait for
the Pod's response.

## Submission and recovery

The runner makes up to 3 passes by default; set `--attempts N` to change the limit.
Each pass opens a witness session and reads submitted results from the Pod.
Any submitted result, including `None`, is skipped by prompt ID before downloading.
Pending prompts call the provided generation function and submit the result with
`witness.submit(image, source)`.
Successful prompts leave the queue; failed prompts are tried on the next pass.
The runner stops when all prompts are submitted and raises if the attempt limit
is reached with unfinished prompts.

Candidate JavaScript, previews and the selected result stay in memory. The miner
does not save result files or use local outputs to decide whether a prompt is done.
Prompt failures are logged with a traceback; other prompts continue.

Unsubmitted prompts run generation again on the next pass or command invocation,
reusing the Pod's retained model responses for identical requests and request IDs.
Preview rendering repeats. Keep the same source, images, seed, models, token
settings, hotkey and authentication mode during the round's access window.
Embedded preview bytes must match for subsequent model requests to reuse recordings.

Each pass uses a native async witness client for model calls, rendering and
submission. Recovery uses the Pod's recordings; the SDK does not write local
recovery files.

## Development

```sh
poetry install
poetry run poe lint
```

Runs Ruff with fixes, Black, Bandit, and strict mypy. It runs all four steps and
exits nonzero if any fail.

## Submission

The root `main.py` exports `async generate(image, seed, witness) -> str | None`.
The CLI passes this function and a dictionary of prompt IDs to sources into the runner.
Sources can be image URLs, file paths or bytes. The generation function uses
`witness.responses.create(...)` for model calls and `witness.render(...)` for previews. Replay supplies recorded model
responses and replaceable rendering through the witness object. Submission and
pending-work checks belong to the live runner.
Commit it with `miner/` and `instructions.md`. The latter contains the Three.js
rules sent to the model. Models and token limits are constants in `miner/pipeline.py`.
