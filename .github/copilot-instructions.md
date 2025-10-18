## Quick orientation for AI coding agents

This file gives focused, actionable knowledge to help an AI agent be productive in the ComfyUI codebase.

1) Big picture
- Entrypoint: `main.py` — parses CLI args, sets environment, loads custom nodes, and starts the asyncio loop that runs the server and prompt worker.
- HTTP + WebSocket server: `server.py::PromptServer` — registers REST routes, static frontend (`app.frontend_management.FrontendManager`) and the `/ws` websocket used by the frontend. First WS message from client negotiates `feature_flags`.
- Execution engine: `execution.py` and `comfy_execution/*` — the prompt worker (spawned in `main.py`) pulls items from `PromptQueue` and runs `PromptExecutor`. Progress updates are sent via `comfy.utils.set_progress_bar_global_hook` (see `start_comfyui()` and `hijack_progress()` in `main.py`).
- Node definitions: `nodes.py` (and `custom_nodes/`) — nodes are classes implementing `ComfyNodeABC` (see `INPUT_TYPES`, `FUNCTION`, `CATEGORY`, `RETURN_TYPES`). Custom nodes are discovered and initialized via `nodes.init_extra_nodes()` and `app.custom_node_manager.CustomNodeManager`.

2) Key files to consult while coding
- `main.py` — startup sequence, env var handling, custom-node prestartup hook (`prestartup_script.py`), and where the asyncio loop / prompt worker is created.
- `server.py` — all HTTP and websocket routes, request middleware (`deprecation_warning`, CORS/origin middleware), and upload/templating routes.
- `nodes.py` — canonical node implementations and examples (e.g. `CLIPTextEncode`). Use as pattern when adding/removing node inputs, returns, or category placement.
- `comfy/cli_args.py` — authoritative list of CLI flags (front-end version/root, `--disable-all-custom-nodes`, `--whitelist-custom-nodes`, device and precision flags, cache flags, preview method, etc.).
- `app/custom_node_manager.py` — custom node conventions: `locales/` structure, example workflow folders (`example_workflows`, `examples`), and how translations are merged.
- `app/frontend_management.py` — how frontend is loaded (local `--front-end-root` vs. remote `--front-end-version`) and the `comfyui-frontend-package` dependency.
- `folder_paths.py` — canonical locations for `models/checkpoints`, `vae`, `loras`, `embeddings`, `custom_nodes`, `input`, `output`, `temp`, and how extra paths are loaded (`extra_model_paths.yaml.example`).

3) Project-specific patterns and gotchas
- Node contract: nodes declare `INPUT_TYPES`, `RETURN_TYPES`, `FUNCTION` and `CATEGORY`. Returning shapes and metadata are common (see `VAEDecode`, `VAEEncode`). Follow existing naming and typing patterns.
- Custom node init: the repo executes `prestartup_script.py` inside custom node folders (if present). Avoid side-effects that assume server state during import — prestartup scripts are explicitly executed by path (see `execute_prestartup_script()` in `main.py`).
- Translations: custom node translations live under `custom_nodes/<node>/locales/<lang>/main.json` and may include `nodeDefs.json`, `commands.json`, `settings.json` (merged by `CustomNodeManager.build_translations`).
- Frontend negotiation: the client sends a `feature_flags` message as the first WS message; server responds with server-side feature flags. Use `comfy_api.feature_flags` utilities when adding new features that need negotiation.
- Torch import timing: do not import `torch` before early initialization in `main.py` — the startup flow expects to configure environment variables and device visibility before importing torch. `main.py` logs a warning if `torch` is already imported.

4) Common developer workflows (discoverable commands)
- Install deps: `pip install -r requirements.txt` (top-level README).
- Run locally: `python main.py` (starts HTTP server + WS at `--listen`/`--port`, default `127.0.0.1:8188`).
- Use frontend from package: set `--front-end-version` or `--front-end-root`. Latest frontend: `--front-end-version Comfy-Org/ComfyUI_frontend@latest`.
- TLS: `--tls-keyfile <key> --tls-certfile <cert>` enables HTTPS.
- Disable custom nodes: `--disable-all-custom-nodes`; allow specific folders with `--whitelist-custom-nodes name1 name2`.
- CI quick smoke: `--quick-test-for-ci` will start early and exit; useful for CI validations.
- Previews: `--preview-method` supports `auto`, `latent2rgb`, `taesd` — see README for TAESD model files and `--preview-size`.
- Tests: pytest configuration exists (`pytest.ini`) — run `pytest` in repo root to execute unit tests.

5) Integration and extension points
- Custom nodes: add a folder under `custom_nodes/<name>/` containing node modules and optional `prestartup_script.py`, `locales/`, and `example_workflows/`.
- API nodes: `comfy_api/` contains API-node support; nodes can be registered via `comfy_api.internal` and versioned through `register_versions` patterns used in `nodes.py`.
- Frontend assets: static web root is resolved by `app.frontend_management.FrontendManager` and served by `PromptServer.web_root`.
- Database (optional): `app/database/` and `--database-url` control DB path/connection; database initialization is attempted at startup if dependencies are present.

6) Safety and non-goals for the agent
- Don't assume runtime GPUs are present — queries to `comfy.model_management.get_torch_device()` and CLI flags (`--cpu`) change behavior.
- Avoid changing or importing optional heavy dependencies at top-level imports (torch, taesd, large model packs) — these are controlled at startup or behind CLI flags.

If anything above is unclear or you want more examples (e.g., a concrete custom-node scaffold or a sample node PR checklist), tell me which section to expand and I'll iterate.
