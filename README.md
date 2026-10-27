# gpu-ai-interactive-sessions-manifest

The GPU model manifest for AI chat GPU sessions. It declares which model environments the `launch_gpu_session` tool in [ai-service](https://github.com/lifebit-ai/ai-service) offers, the container image each one runs in, how each model is invoked, and which EC2 instance types the whole set has been validated on.

The manifest is data, not code: adding a model, changing its guidance or validating a new instance type is a change to `manifest.json` here, with no ai-service release.

## How ai-service loads it

- **Where from.** `GPU_SESSION_MODEL_MANIFEST_URI` in each ai-service deployment points at this file, usually the raw GitHub URL of `manifest.json` on `main`. ai-service accepts an `https://` URL or a local file path; `s3://` is refused.
- **When.** Only when `ENABLE_GPU_SESSION_TOOL` is on. The first chat turn that binds or lists tools fetches it (10 s timeout), validates it, and caches it for the life of the ai-service process.
- **Changes need a restart.** A merge here reaches a deployment only when its ai-service pods restart, because the cached copy is never refreshed. Since the URL tracks `main`, every merge is picked up by every deployment at its next restart.
- **Failure is contained.** If the manifest is unset, unreachable or invalid, `launch_gpu_session` is not offered and an error is logged under `[GpuModelManifest]`. A failed load is retried by a later turn after a 60 s cooldown. Every other tool, including `launch_interactive_session`, is unaffected.
- **Tool availability.** The tool is offered only when the manifest is loaded and every instance type in the deployment's `GPU_SESSION_INSTANCE_TYPES` is listed in `validatedInstanceTypes`. If even one is missing, the tool stays hidden and the load logs an error naming the unvalidated types.
- **Instance-type fallback.** `GPU_SESSION_INSTANCE_TYPES` is an ordered, comma-separated list, e.g. `g6e.2xlarge,g7e.2xlarge`. A new GPU session launches on the first type; if that launch fails, ai-service resumes the aborted session on the next type, until one comes up or the list runs out.
- **Unknown keys are ignored.** The schema strips fields it does not know, so a new field can be added here ahead of the ai-service release that reads it.

The schema lives in ai-service at `src/core/common-tools/execute-code-gpu-session/model-manifest/schemas.ts`.

## Top-level fields

| Field | Required | Purpose | Used by ai-service |
|---|---|---|---|
| `manifestVersion` | yes | Identifies this revision, e.g. `2026-10-26-r2`. Bump it on every change. | Logged on load, reported in each GPU tool result and launch log, and named in errors. |
| `validatedInstanceTypes` | yes, at least one | EC2 instance types the GPU AMI and every model in this manifest have been tested on. Currently `g6e.2xlarge` and `g7e.2xlarge`. | Gates the tool: every type in the deployment's `GPU_SESSION_INSTANCE_TYPES` must be listed. The order that matters is the one in `GPU_SESSION_INSTANCE_TYPES`, not this list. An existing chat session on a type not listed is refused rather than reused, and an aborted session's last type is retried first on resume only while it is still listed. |
| `models` | yes, at least one | The model environments the tool offers. Names must be unique. | See below. |
| `workflow` | no | How the models fit together, in order. | Appended to the tool's `model` parameter description as guidance for the LLM. |

## Model fields

| Field | Required | Purpose | Used by ai-service |
|---|---|---|---|
| `name` | yes | Model identifier, e.g. `boltz2`. | Becomes one value of the tool's `model` enum, which the LLM picks per call. |
| `role` | yes | Short description of what the model is for. Free text. | Shown to the LLM next to the name. |
| `image` | yes | Bare image repository name, without registry host or tag. | Sent to api-server as `image@imageDigest` for the execution. |
| `imageDigest` | yes | `sha256:` digest of the validated image. | Pins the execution to exactly that image, even if the tag moves. The image is baked into the GPU AMI and run without pulling, so a new digest also needs a new AMI. |
| `mode` | yes | Kit mode: `fast`, `exact` or `off`. | Sent with the execution and exposed in the container as `LIFEBIT_MODEL_MODE`. The LLM fills the `{mode}` placeholder from it. |
| `modeReason` | no | Why this mode was chosen on the validated GPUs. | Validated only; not shown to the LLM. Documentation for whoever changes the mode. |
| `extraFlags` | yes, may be empty | Extra kit flags validated for this model. | Sent with the execution and exposed as a JSON array in `LIFEBIT_MODEL_EXTRA_FLAGS`. The LLM fills `{extraFlags}` from it. |
| `minComputeCapability` | no | Minimum GPU compute capability. | Validated only; not enforced. Keeping it true across `validatedInstanceTypes` is the manifest owner's job. |
| `minDriver` | yes | Minimum NVIDIA driver version. | Validated only; not enforced. The GPU AMI must ship a driver at least this new. |
| `weights` | no | Model weights to install on first use. Omit for models that need none, like `general`. | See below. |
| `workDir` | yes | Directory to run the kit command from. | Shown to the LLM in the model description. |
| `entrypointHint` | yes | Command template for running the model. Carries no mode or flag values, only the `{mode}` and `{extraFlags}` placeholders where the kit takes them; other `{placeholders}` are for the LLM to fill from the task. | Shown to the LLM in the model description. |
| `guidance` | no | Model-specific instructions: inputs, checks, pitfalls. | Appended to the model description for the LLM. |

### `weights`

| Field | Required | Purpose | Used by ai-service |
|---|---|---|---|
| `installCommand` | yes | Command that installs the weights into the session. | Shown to the LLM as the step to run before the model's first use in a session. Weights stay in `/weights` for the rest of the session. |
| `path` | yes | Where the weights live in the container. | Validated only. |
| `envVar` | yes | Environment variable the kit reads the weights location from. | Validated only. Nothing sets it in the container, so the image must. |
| `approxSizeGb` | yes | Approximate download size. | Validated only. |
| `mirrorPrefix` | yes | Prefix of the weights in a Lifebit mirror. | Validated only. A mirror is not wired yet: `GPU_SESSION_WEIGHTS_SOURCE` and `GPU_SESSION_WEIGHTS_MIRROR_URI` exist in ai-service config but nothing reads them. |

## Changing the manifest

- Bump `manifestVersion` on every change.
- Add an instance type to `validatedInstanceTypes` only after the GPU AMI and every model have run on it. A deployment can then add it to `GPU_SESSION_INSTANCE_TYPES`.
- Before removing an instance type, take it out of `GPU_SESSION_INSTANCE_TYPES` in every deployment first, or the tool disappears there at the next restart.
- A new image or digest must be baked into the GPU AMI before the manifest points at it.
- Keep `entrypointHint` free of mode and flag values; `mode` and `extraFlags` are the only source of them.
- Validate the JSON before merging, and restart the ai-service pods of each deployment that should pick up the change.
