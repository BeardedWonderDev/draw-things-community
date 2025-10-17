# LoRA Training Scripting Integration Plan

## Context
- Target workflow: enable an external Swift helper application to trigger LoRA training inside a compiled Draw Things build without embedding Draw Things source.
- Current interfaces: Draw Things exposes JavaScript-based automation through `Libraries/Scripting/ScriptExecutor.swift`, while the public HTTP server (`Libraries/HTTPAPIServer`) only supports inference routes.
- Training implementation: LoRA training pipelines already exist in `Libraries/Trainer/LoRATrainer.swift` and are used by the shipping UI, but are not reachable over scripting or HTTP.

## Goal
Expose a script-accessible entry point that wraps existing LoRA training capabilities so an external controller can provide dataset paths, configuration, and receive progress or failure updates when training runs inside the compiled app.

## Proposed Enhancements
1. **Scripting Bridge Extension**
   - Add `trainLoRA(_:)` (name TBD) to the `JSInterop` protocol and `ScriptExecutor` so JavaScript can submit training jobs.
   - Extend `ScriptExecutorDelegate` with methods to kick off training and surface progress back to JavaScript (`log`, structured callbacks, or promises).
2. **Delegate Implementation**
   - In the app target, implement the new delegate methods by preparing datasets and invoking `LoRATrainer.train`.
   - Persist checkpoints using existing `LoRATrainerCheckpoint` utilities and forward per-step progress by reusing the `progressHandler`.
3. **Script-Side API**
   - Update `SharedScript.swift` template to expose `pipeline.trainLoRA({...})`, validating parameters and normalizing paths.
   - Document script payload schema (dataset descriptors, configuration overrides, callback hooks).
4. **Automation Flow for Helper App**
   - Helper app writes a JavaScript payload, uploads or references dataset files on disk, executes the script via the Draw Things scripting interface, and listens for console output or structured events.
5. **Validation**
   - Provide sample scripts and a minimal dataset to verify end-to-end training, including resume/abort scenarios.

## Open Questions / Risks
- Progress feedback channel: choose between console logs, explicit callbacks, or temporary files.
- Long-running job management: define cancellation semantics for scripting (e.g., `pipeline.cancelTraining()`).
- Dataset accessibility: ensure the compiled app can reach file paths provided by the helper app (sandboxing on macOS/iOS).
- Resource contention: training is GPU-intensive; document scheduling rules if inference and training run concurrently.

## Success Criteria
- External scripting call launches training, emits structured progress, and produces LoRA checkpoints accessible to the helper app.
- Failure cases (missing assets, invalid configuration, interruption) return actionable errors through the scripting channel.
- Documentation updated so contributors understand the scripting-based training workflow and integration pattern.
