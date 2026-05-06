# sim-plugin-hfss

Use Codex, Claude Code, or another AI agent to work with
[Ansys HFSS](https://www.ansys.com/products/electronics/ansys-hfss) 3D
projects through [sim-cli](https://github.com/svd-ai-lab/sim-cli).

`sim-plugin-hfss` is an initial HFSS 3D driver plugin for sim-cli. It uses
PyAEDT as the Python control layer for Ansys Electronics Desktop (AEDT), keeps
the driver import-safe on machines without AEDT, and bundles an HFSS agent
skill so an agent has solver-specific workflow guidance after installation.

The HFSS/AEDT application and license are not bundled. Bring your own AEDT
installation and license. See [LICENSE-NOTICE.md](LICENSE-NOTICE.md).

## Current maturity

This is a bootstrap release candidate. It has unit coverage, protocol
conformance coverage, simulated PyAEDT session coverage, and packaging checks.
It has not yet been run against a real HFSS installation because the initial
development machine does not have licensed AEDT/HFSS available.

Use it as an integration starting point, not as proof that a production HFSS
workflow has been validated end to end.

## Scope

Version 0.1.0 targets HFSS 3D through PyAEDT's `ansys.aedt.core.hfss.Hfss`
interface.

Out of scope for this first version:

- HFSS 3D Layout
- Maxwell, Icepak, Q3D, Circuit, or generic AEDT workflows
- Direct `.aedt` or `.aedtz` batch solve without a PyAEDT script
- PyPI publishing or plugin-index catalogue entry

## What an agent can do with HFSS

- Detect PyAEDT Python scripts that instantiate HFSS.
- Check whether AEDT appears to be installed on the host.
- Start a PyAEDT-backed HFSS session in graphical or non-graphical mode when
  AEDT is available.
- Execute bounded Python snippets against the active `hfss` object.
- Inspect session, project, and design summaries before continuing.
- Run complete PyAEDT Python scripts through `sim run --solver hfss`.

## Install

For source testing:

```bash
uv pip install "git+https://github.com/svd-ai-lab/sim-plugin-hfss.git@main"
```

After installation, sim-cli should auto-discover the driver and bundled skill:

```bash
sim check hfss
sim run --solver hfss path/to/script.py
```

If `sim check hfss` reports that AEDT itself is unavailable, first confirm the
Python package installed correctly, then fix the local AEDT installation,
environment variables, or license prerequisites.

## AEDT discovery

The driver looks for AEDT using:

- `SIM_HFSS_AEDT_ROOT`
- `SIM_AEDT_ROOT`
- `ANSYSEM_ROOT*`
- `ansysedt` or `ansysedt.exe` on `PATH`
- conservative default Windows and Linux install paths

If AEDT is installed in a nonstandard location, set an explicit root:

```powershell
$env:SIM_HFSS_AEDT_ROOT = 'C:\Program Files\AnsysEM\v261\Win64'
sim check hfss
```

## Common agent workflow

1. Run `sim check hfss`.
2. Choose GUI mode only when visual review is required; otherwise prefer
   non-graphical mode.
3. Connect and inspect the active project/design before mutating anything:

   ```bash
   sim connect --solver hfss --ui-mode no_gui
   sim inspect session.summary
   sim inspect hfss.project.identity
   sim inspect hfss.design.summary
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
smoke testing must be added once a licensed AEDT installation is available.

## License

Apache-2.0. See [LICENSE](LICENSE) and [LICENSE-NOTICE.md](LICENSE-NOTICE.md).
