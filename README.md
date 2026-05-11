# sim-plugin-hfss

Use Codex, Claude Code, or another AI agent to work with
[Ansys HFSS](https://www.ansys.com/products/electronics/ansys-hfss) 3D
projects through [sim-cli](https://github.com/svd-ai-lab/sim-cli).

`sim-plugin-hfss` is an initial HFSS 3D driver plugin for sim-cli. It uses
PyAEDT as the Python control layer for Ansys Electronics Desktop (AEDT), keeps
the driver import-safe on machines without AEDT, and bundles an HFSS agent
skill so an agent has solver-specific workflow guidance after installation.

The HFSS/AEDT application is not bundled. See
[LICENSE-NOTICE.md](LICENSE-NOTICE.md).

## Current maturity

This is an initial alpha release. It has unit coverage, protocol conformance
coverage, simulated PyAEDT session coverage, packaging checks, and opt-in real
HFSS smoke coverage for hosts with AEDT available.

Use it as an integration starting point, not as proof that a production HFSS
workflow has been validated end to end.

## Scope

Version 0.1.0 targets HFSS 3D through PyAEDT's `ansys.aedt.core.hfss.Hfss`
interface.

Out of scope for this first version:

- HFSS 3D Layout
- Maxwell, Icepak, Q3D, Circuit, or generic AEDT workflows
- Direct `.aedt` or `.aedtz` batch solve without a PyAEDT script
- Plugin-index catalogue entry before the package is published and smoke-tested

## What an agent can do with HFSS

- Detect PyAEDT Python scripts that instantiate HFSS.
- Check whether AEDT appears to be installed on the host.
- Start a PyAEDT-backed HFSS session in graphical or non-graphical mode when
  AEDT is available.
- Execute bounded Python snippets against the active `hfss` object.
- Inspect session, project, and design summaries before continuing.
- Run complete PyAEDT Python scripts through `uv run sim run --solver hfss`.

## Install

For agent projects, install sim-cli-core and the HFSS plugin in the project
environment:

```powershell
uv init  # only if this is not already a uv project
uv add sim-cli-core sim-plugin-hfss
uv run sim plugin sync-skills --target .agents/skills --copy
uv run sim check hfss
uv run sim plugin doctor hfss --deep
```

For Claude Code, sync the bundled skill to `.claude/skills` instead:

```powershell
uv run sim plugin sync-skills --target .claude/skills --copy
```

For source testing against the current main branch:

```powershell
uv add sim-cli-core "git+https://github.com/svd-ai-lab/sim-plugin-hfss.git@main"
```

`uv run sim ...` runs sim from this project environment, so it sees this
project's plugins. Without uv, create and activate a venv, then install
`sim-cli-core` plus this plugin with `python -m pip`.

If `uv run sim check hfss` reports that AEDT itself is unavailable, first
confirm the Python package installed correctly, then fix the local AEDT
installation, environment variables, or runtime prerequisites.

## AEDT discovery

The driver looks for AEDT using:

- `SIM_HFSS_AEDT_ROOT`
- `SIM_AEDT_ROOT`
- `ANSYSEM_ROOT*`
- AEDT launchers such as `ansysedt`, `ansysedt.exe`, or `ansysedtsv.exe` on
  `PATH`
- conservative default Windows and Linux install roots

If AEDT is installed in a nonstandard location, set an explicit root:

```powershell
$env:SIM_HFSS_AEDT_ROOT = 'C:\path\to\AnsysEM'
uv run sim check hfss
```

You do not need to add AEDT to the global system `PATH` when default discovery
or one of the explicit environment variables works.

## Common agent workflow

1. Run `uv run sim check hfss`.
2. Choose GUI mode only when visual review is required; otherwise prefer
   non-graphical mode.
3. Connect and inspect the active project/design before mutating anything:

   ```bash
   uv run sim connect --solver hfss --ui-mode no_gui
   uv run sim inspect session.summary
   uv run sim inspect hfss.project.identity
   uv run sim inspect hfss.design.summary
   ```

4. Run one bounded PyAEDT snippet at a time.
5. Inspect `last.result` and design state before solving or exporting.
6. Validate engineering results from HFSS artifacts and domain criteria, not
   from process success alone.

## Develop

```bash
git clone https://github.com/svd-ai-lab/sim-plugin-hfss
cd sim-plugin-hfss
uv sync --extra test
uv run pytest -q
uv build
```

The test suite is designed to pass on machines without AEDT/HFSS. Real solver
smoke testing is opt-in:

```bash
SIM_HFSS_RUN_INTEGRATION=1 uv run pytest tests/test_hfss_real_smoke.py -q
```

On PowerShell:

```powershell
$env:SIM_HFSS_RUN_INTEGRATION = '1'
uv run pytest tests/test_hfss_real_smoke.py -q
```

## License

Apache-2.0. See [LICENSE](LICENSE) and [LICENSE-NOTICE.md](LICENSE-NOTICE.md).
