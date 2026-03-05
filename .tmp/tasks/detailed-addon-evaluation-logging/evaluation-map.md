# Subtask 01 - Evaluation Map

Feature: detailed-addon-evaluation-logging
Scope: map evaluation points, identify logging ambiguity, propose canonical event shape.

## Key Evaluation Points (Current Code)

1. `ops_transcribe.py::_BaseTranscribeOperator.invoke`
   - Evaluates: active job lock, selected strip, filepath validity, runtime config build.
   - Current output: mostly `self.report(...)` UI messages; little console-level diagnostic detail.

2. `ops_transcribe.py::_BaseTranscribeOperator._drain_queue`
   - Evaluates: worker message types (`progress`, `error`, `complete`, `cancelled`) and state transitions.
   - Current output: updates UI properties; no structured console trail of transitions.

3. `ops_transcribe.py::_BaseTranscribeOperator._transcribe_worker`
   - Evaluates: model load readiness, extraction path, optional vocal separation, VAD retry fallback decisions.
   - Current output: progress text plus final error string; decision branches (low recall fallback, retry wins/losses) are mostly opaque in console.

4. `core/transcriber.py::TranscriptionManager.load_model`
   - Evaluates: device selection, CUDA runtime prep, local model integrity, compute-type compatibility.
   - Current output: `print(...)` messages for major failures/warnings; partially informative, not consistently structured.

5. `core/transcriber.py::TranscriptionManager.transcribe`
   - Evaluates: option assembly, detected language, segment processing progress.
   - Current output: progress callback text, but no stable event IDs or normalized context fields.

6. `ops_dependencies.py::SUBTITLE_OT_check_dependencies.execute`
   - Evaluates: interpreter paths and dependency import checks.
   - Current output: verbose console prints (good visibility), but inconsistent formatting and no explicit pass/fail event object.

7. `ops_dependencies.py::SUBTITLE_OT_install_pytorch._install_thread`
   - Evaluates: backend selection path, command construction, install result, CUDA runtime package follow-up.
   - Current output: command prints plus status updates; lacks consistent phase/outcome/reason structure.

8. `props.py::SubtitleEditorProperties.update_text`
   - Evaluates: edit target resolution and strip sync.
   - Current output: only unresolved-target print; successful/no-op paths have no evaluation logs.

## Ambiguous or Missing Console Outputs

1. Silent state transitions in modal flow
   - Location: `ops_transcribe.py::_drain_queue`, `_finalize`, `_cleanup`
   - Gap: queue events and final outcome paths are not logged as structured transitions.
   - Effect: hard to explain what happened when behavior is wrong but no exception is raised.

2. VAD fallback decisions are under-explained
   - Location: `ops_transcribe.py::_transcribe_worker`
   - Gap: low-recall detection, relaxed VAD retry, and no-VAD retry decisions are not consistently logged with input metrics and chosen branch.
   - Effect: difficult to understand why transcript quality changed or remained poor.

3. Model readiness diagnostics are not normalized
   - Location: `core/transcriber.py::load_model`
   - Gap: errors are printed as free text; no stable fields for action, phase, or reason code.
   - Effect: inconsistent debugging and harder log filtering.

4. Dependency checks are verbose but non-uniform
   - Location: `ops_dependencies.py::SUBTITLE_OT_check_dependencies.execute`
   - Gap: prints are human-readable but not machine-parsable or standardized across operators.
   - Effect: difficult to compare outcomes across sessions.

5. Success/no-op paths often have no console evidence
   - Location: multiple operator/property update paths
   - Gap: only failures/warnings tend to be printed.
   - Effect: second-guessing when users report unexpected behavior without stack traces.

## Proposed Canonical Log Event Shape

Use one compact, stable line format for all evaluation logs:

```text
[Subtitle Studio][EVAL] action=<action> phase=<phase> outcome=<outcome> reason=<reason> context=<json>
```

Required fields:
- `action`: logical operation (`transcribe.invoke`, `transcribe.worker`, `deps.check`, `model.load`)
- `phase`: `start|checkpoint|decision|success|warning|fail|cancel`
- `outcome`: `ok|warn|error|cancelled|noop`
- `reason`: short code-like reason (`missing_strip`, `low_recall_retry`, `model_not_found`)
- `context`: sanitized JSON payload (small, non-sensitive)

Recommended context keys:
- `scene`, `strip_name`, `model`, `device`, `compute_type`
- `audio_duration`, `segments`, `word_count`, `coverage`
- `command_hint` (safe short command summary only)

Sanitization rules:
- No absolute user paths unless needed for diagnosis.
- No secrets/tokens.
- Truncate long values.

## Minimum Coverage Targets for Subtask 03

- `transcribe.invoke`: start/fail checkpoints for strip + filepath validation.
- `transcribe.worker`: model load, retry decisions, completion stats.
- `model.load`: path/device/compute decision + normalized failure reason.
- `deps.check`: summary event with missing dependency list and GPU detection outcome.
- `deps.install`: install command phase + return code outcomes.
