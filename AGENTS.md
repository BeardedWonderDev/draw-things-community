# Repository Guidelines
## Project Structure & Module Organization
Draw Things Community is a Bazel workspace. Core Swift modules that mirror app functionality live in `Libraries/` (e.g., `Libraries/SwiftDiffusion`, `Libraries/GRPC`, `Libraries/ImageGenerator`). Executable entry points live in `Apps/`: the converters and quantizer targets share code, while `Apps/gRPCServerCLI` packages the server and `Apps/macOS`/`Apps/Linux` wrap platform-specific bundles and Docker images. Developer automation lives in `Scripts/` (pre-commit hooks, build utilities, Docker helpers), with vendored or mirrored dependencies kept under `external/` and `Vendors/`. Platform packaging helpers sit in `Tools/` and `Apps/*/SupportingFiles`.

## Build, Test, and Development Commands
- `./Scripts/install.sh` installs Bazel workspace links and sets up pre-commit hooks (run once per clone, rerun after Bazel config changes).
- `bazel build Apps:gRPCServerCLI` compiles the CLI; append `-c opt --macos_minimum_os=13.0` as needed for release builds.
- `bazel build Apps:ModelConverter` (or `LoRAConverter`, `ModelQuantizer`, etc.) produces the command-line tools mirrored in the app.
- `bazel run Apps:ModelConverter -- --help` is a quick smoke check that a freshly built binary executes.
- `bazel build Apps/Linux:push_gRPCServerCLI_image` builds and stages the Docker image used for the CUDA server.

## Coding Style & Naming Conventions
Swift sources use `swift-format` (2-space indents, 100-column limit, lowerCamelCase) with rules defined in `.swift-format.json`. Install hooks with `./Scripts/install.sh`; otherwise format manually via `bazel run @SwiftFormat//:swift-format -- format --configuration .swift-format.json path/to/File.swift`. Bazel files must pass `buildifier`, automatically invoked by the hook; keep target names UpperCamelCase and group deps alphabetically. Shell scripts are POSIX sh/bash with `set -euo pipefail` where practical.

## Testing Guidelines
Run `bazel test //...` before posting a PR; it is lightweight today but catches interface regressions as tests land. For end-to-end validation of model serving, execute `ruby Scripts/tests/test_cli_and_server.rb bazel-bin/Apps/gRPCServerCLI /path/to/models $(pwd)` after building the CLI, ensuring sample inputs (e.g., `Scripts/tests/dog.png`) produce consistent output. Document any long-running or GPU-required checks in your PR.

## Commit & Pull Request Guidelines
The history favors concise, sentence-style subject lines (e.g., `Add LoRA import support.`). Include the behavior change, the affected module, and why. Each PR should describe how to reproduce the result, list the Bazel targets you built/tested, and attach screenshots or sample output for UI/server changes. Link relevant issues, confirm the CLA is signed (`cla.yaml` enforces this), and avoid force-pushing once reviews begin so internal sync jobs can mirror your branch cleanly.
