# LoRA Training gRPC Integration Plan

## Context
- **Objective**: Allow external applications to trigger and monitor LoRA training inside the compiled Draw Things environment without bundling Draw Things source code.
- **Current capabilities**: The public `gRPCServerCLI` exposes only image-generation RPCs (`ImageGenerationService` in `Libraries/GRPC/Models/Sources/imageService.proto`) while LoRA training lives in `LoRATrainer` within the app code path.
- **Constraints**: Reuse the existing remote infrastructure (Swift gRPC, shared-secret auth, file upload RPC) and avoid relying on the UI-specific JavaScript scripting bridge.

## High-Level Approach
Add a dedicated `LoRATrainingService` gRPC API beside `ImageGenerationService`. External clients will:
1. Upload or reference training assets.
2. Call `StartTraining` with a serialized `LoRATrainingConfiguration` and dataset metadata.
3. Stream progress via `SubscribeTraining` (server-streaming).
4. Receive completion metadata (checkpoint paths, metrics) or failure diagnostics.

## Protocol Additions
- Extend `Libraries/GRPC/Models/Sources/imageService.proto` (or add `loratraining.proto`) with:
  - `message StartTrainingRequest { LoRATrainingConfiguration config; repeated TrainingExample examples; string outputDirectory; string jobId; optional string sharedSecret; }`
  - `message TrainingExample { string imagePath; string caption; optional string maskPath; }`
  - `message TrainingProgress { string jobId; enum Stage { COMPILE, STEP, CHECKPOINT, COMPLETE, FAILED, CANCELLED }; float percentComplete; string message; optional LoRATrainerCheckpoint checkpoint; }`
  - `message StartTrainingResponse { string jobId; }`
  - RPCs:
    - `rpc StartTraining(StartTrainingRequest) returns (StartTrainingResponse);`
    - `rpc SubscribeTraining(TrainingStatusRequest) returns (stream TrainingProgress);`
    - `rpc CancelTraining(TrainingCancelRequest) returns (TrainingCancelResponse);`
    - `rpc ListTrainingJobs(TrainingJobsRequest) returns (TrainingJobsResponse);` (optional dashboarding)
- Regenerate Swift proto stubs via Bazel (`//Libraries/GRPC:Models`).

## Server Implementation
- Create `LoRATrainingServiceImpl` in `Libraries/GRPC/Server/Sources/`:
  - Wrap `LoRATrainer.prepareDataset` / `train` (`Libraries/Trainer/Sources/LoRATrainer.swift:749` & `:1618`).
  - Maintain a job manager that tracks state, progress callbacks, cancellation tokens, and checkpoint outputs.
  - Stream progress by translating trainer callbacks (`TrainingState.compile`, `.step`, etc.) into `TrainingProgress` messages.
  - Persist checkpoints under a configurable workspace directory using `LoRATrainerCheckpoint`.
- Update `Apps/gRPCServerCLI/gRPCServerCLI.swift` to register the new service with the existing `ServerConfigurationRewriter`/queue setup.
- Add CLI flags/env vars for:
  - Enabling/disabling training RPCs.
  - Allowed dataset roots/output directories.
  - Maximum concurrent jobs (default 1 due to GPU pressure).

## Client Workflow
1. Optional: use existing `UploadFile` RPC to push dataset images/captions to the server.
2. Call `StartTraining` with job metadata; store returned `jobId`.
3. Open `SubscribeTraining(jobId)` to receive streamed progress (compile, per-step %, checkpoint created).
4. On completion, download checkpoints if needed or use server-specified path.
5. Invoke `CancelTraining(jobId)` to stop jobs early.

## Testing & Validation
- Unit-test the job manager (queueing, cancellation, checkpoint retention).
- Integration tests via Bazel:
  - Launch `gRPCServerCLI` in-process, start a toy dataset training job, verify streamed progress and checkpoint creation.
  - Confirm shared-secret enforcement and error codes (invalid config, missing files).
- Provide sample client script (Swift or Python) under `Scripts/tests/` to demonstrate end-to-end usage.

## Risks & Mitigations
- **GPU contention**: enforce single-job execution and document resource requirements.
- **Large asset transfer**: rely on chunked `UploadFile` RPC, add checksum validation in the server.
- **Backward compatibility**: guard registration behind a feature flag so existing deployments remain inference-only by default.

## Deliverables
- Updated proto definitions and generated Swift stubs.
- New `LoRATrainingServiceImpl` and job-management utilities.
- CLI updates (flags, documentation).
- Example client and README/AGENTS addendum explaining the remote training workflow.
