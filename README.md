# 9r-qoder-patch

> ### Need a Qoder account?
>
> **Base 400 credits** Qoder accounts — order via Telegram:
> **[@contools_bot](https://t.me/contools_bot)**
>
> Third-party seller, not affiliated with this repo or with 9router. Check the
> offer yourself before paying.

Three fixes for **Qoder (`qd/*`) in 9router**.

Run it from the root of your 9router repo. No install, no environment variables,
no build step — it edits seven source files and five test files in place, and
`npm run dev` / `npm run build` pick them up as usual.

---

## Run it

### Linux / macOS / WSL / Git Bash

```bash
bash patch.sh
```

Or without saving the file first:

```bash
curl -fsSL https://raw.githubusercontent.com/capncodes69/9r-qoder-patch/main/patch.sh | bash
```

### Windows (PowerShell)

```powershell
.\patch.ps1
```

Or one line, without saving the file:

```powershell
irm https://raw.githubusercontent.com/capncodes69/9r-qoder-patch/main/patch.ps1 | iex
```

> PowerShell blocks unsigned scripts by default. If `.\patch.ps1` refuses to run,
> use the `irm | iex` form above, or allow it for this session:
> ```powershell
> Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
> ```

### cmd.exe

```cmd
powershell -NoProfile -Command "irm https://raw.githubusercontent.com/capncodes69/9r-qoder-patch/main/patch.ps1 | iex"
```

---

## Check / revert

| | Linux, macOS, WSL, Git Bash | Windows PowerShell |
| --- | --- | --- |
| Apply | `bash patch.sh` | `.\patch.ps1` |
| Check | `bash patch.sh --check` | `.\patch.ps1 -Check` |
| Revert | `bash patch.sh --revert` | `.\patch.ps1 -Revert` |

`--check` exits `0` when patched, `1` when not patched, and `2` when the patch
would not apply (upstream moved). Applying twice is a no-op, not an error.

The `irm | iex` form cannot take parameters, so check and revert use environment
variables instead:

```powershell
$env:NINEROUTER_CHECK=1;  irm https://raw.githubusercontent.com/capncodes69/9r-qoder-patch/main/patch.ps1 | iex
$env:NINEROUTER_REVERT=1; irm https://raw.githubusercontent.com/capncodes69/9r-qoder-patch/main/patch.ps1 | iex
```

---

## Requirements

| | Needed for |
| --- | --- |
| **git** | applying and reverting the patch |
| bash | `patch.sh` only — present by default on Linux/macOS, via Git Bash or WSL on Windows |
| PowerShell 5.1+ | `patch.ps1` only — built into Windows 10/11 |

Nothing else. Node.js is **not** required by the script — you already have it if
you can run 9router.

---

## What it fixes

### 1. A queued model is not a billing block

Every `qd/*` request used to answer `403` in about two seconds:

```
HTTP 403  {"code":"403","message":"...10605...isQueued:true..."}
```

Code `10605` is a **capacity** signal, not a quota block. The model is briefly
out of slots and becomes available again in ~30s, but 9router classified it as
billing, so the frame became a `403`, which chatCore reads as
`insufficient_quota` and locks the model for minutes.

This patch waits it out instead, the way qodercli does — poll the queue-status
endpoint and replay once the model is ready:

```
inference  -> 10605 isQueued:true waitTime:30
poll 1     -> isQueued:true  waitTime:28
poll 2     -> isQueued:true  waitTime:12
poll 3     -> isQueued:false waitTime:0
replay     -> 200, real completion
```

Same request, same account: **403 in 1.8s before, 200 with `"PONG"` in 34s after.**

Budget defaults to 3 minutes, raisable with `QODER_MODEL_QUEUE_MAX_WAIT_MS`. When
it runs out you get a retryable `503` (`code: qoder_model_queued`) rather than a
false quota error, so account fallback still rotates.

### 2. Sonus and Cantus are routable

`/algo/api/v2/model/list` is per-account, and the frontier reasoning pair Sonus
(`smodel`) / Cantus (`cmodel`) is frequently absent from it even for accounts
entitled to use them. 9router refused before sending anything:

```
qoder: model_config for "smodel" not yet known
```

The chat endpoint does not validate `model_config.key` against the catalog, so
the patch supplies the block itself. A live catalog entry always wins, and a
genuinely unknown key still throws — a typo cannot silently become a no-op
upstream.

### 3. The `qoder` capability table was missing

`PROVIDER_CAPABILITIES["qoder-cn"]` pointed at a table that did not exist. Qoder's
ids are opaque (`dmodel`, `kmodel`, `qmodel_38max`) and match no name heuristic,
so every Qoder model was reported text-only and an image sent by a client was
swapped for a placeholder before reaching the model.

The patch restores the table. The tier aliases (`auto`, `efficient`, `lite`) stay
text-only because they resolve to a variable target upstream.

---

## What it changes

```
open-sse/executors/qoder.js             queue wait, billing split, stable machineId
open-sse/shared/qoder/queue.js          NEW — queue payload parsing + status polling
open-sse/shared/qoder/constants.js      Sonus/Cantus static model_config blocks
open-sse/shared/qoder/cosy.js           machineId derived from userId
open-sse/services/qoderModels.js        keep static-only keys in /v1/models
open-sse/providers/registry/qoder.js    advertise Sonus / Cantus
open-sse/providers/capabilities.js      restore the qoder capability table

tests/unit/qoder-billing.test.js        UPDATED — 10605 is not a billing block
tests/unit/qoder-context-tier.test.js   UPDATED — static keys lead the model list
tests/unit/qoder-queue.test.js          NEW
tests/unit/qoder-queue-stream.test.js   NEW
tests/unit/qoder-static-models.test.js  NEW
```

Twelve files. No dependencies, no config, no data, no schema change.

**The test files are included on purpose.** Two existing suites assert the
behaviour this patch changes — `qoder-billing` expects `10605` to be a billing
block, `qoder-context-tier` expects the live catalog first. Shipping the source
without them leaves those suites red, which looks like a regression when it is
the intended fix. With them the suite matches what upstream sees on
[#5051](https://github.com/decolua/9router/pull/5051).

## If the patch does not apply

It stops and tells you. That means upstream moved and the context no longer
matches — a partial application would leave one fix present and another missing,
which is harder to diagnose than a clean failure.

The bundle is written against upstream `87a74e1b` (v0.5.104).

## Notes and limits

- **Downstream patch, not an upstream merge.** Submitted as
  [decolua/9router#5051](https://github.com/decolua/9router/pull/5051) and still
  open. When it merges, stop using this.
- **Long queues still fail.** `dmodel` and `mmodel` ran past 250 seconds in
  testing and hit the default budget. Raise `QODER_MODEL_QUEUE_MAX_WAIT_MS` if
  your clients can wait that long.
- **`lite` is not fixed.** It is absent from the catalog too, and there is no
  verified `model_config` for it, so no static block is fabricated.
- **`smodel` / `cmodel` output cannot be attributed.** Every response reports
  `"model":"auto"` regardless of the key sent, so the patch proves the key is
  accepted and a completion streams — not that the output came from Sonus rather
  than `auto`.
- **Not tested:** the `qoder-cn` region (shares the code paths and the capability
  table) and `api3.qoder.sh` as chat host.

## Layout

```
patch.sh                      apply / revert / check  (Linux, macOS, WSL, Git Bash)
patch.ps1                     apply / revert / check  (Windows PowerShell)
patches/9r-qoder-fix.patch    the bundle: 7 files
```

## License

MIT
