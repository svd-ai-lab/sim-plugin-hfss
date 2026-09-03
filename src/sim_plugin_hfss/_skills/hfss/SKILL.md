---
name: hfss
description: "Work with Ansys HFSS 3D through PyAEDT, AEDT, sim-plugin-hfss, or solver-native workflows. Use when the user asks an agent to inspect, build, edit, run, monitor, export, or debug HFSS 3D models."
---

# HFSS Skill

Use this skill for Ansys HFSS 3D work.

This plugin targets HFSS 3D through PyAEDT and AEDT. It does not yet cover HFSS
3D Layout, Maxwell, Icepak, Q3D, Circuit, or generic AEDT workflows.

`sim-cli` is a control and observability layer, not the only valid execution
path. Use `sim check`, `sim connect`, `sim inspect`, and bounded `sim exec`
snippets when they add discovery, session control, or structured evidence. Use
plain PyAEDT scripts, AEDT executables, vendor batch flows, or GUI operation
when those are the narrower reliable primitive. The evidence standard is the
same for every path.

## Required Protocol

1. Run an HFSS/AEDT availability probe before launch or solve.
   - Use `sim check hfss` when `sim-cli` is available.
   - Acceptable alternatives: a PyAEDT import/version probe, AEDT executable
     path probe, Windows Registry/default-install probe, or environment variable
     probe.
   - When probing nested Python packages such as `ansys.aedt.core`, catch
     `ModuleNotFoundError`; a missing top-level `ansys` package is evidence that
     the control package is absent, not a reason to launch AEDT for discovery.
   - Do not use `-help` on `ansysedt.exe`, `ansysedtsv.exe`, or `hfss.exe` as
     an availability probe. AEDT Student can open a modal `Electronics Desktop
     Student Help` window and block automation when launched this way.
   - Missing `sim-cli` is not evidence that AEDT/HFSS is missing.
2. If no AEDT installation is found, stop and ask for an AEDT installation or
   `SIM_HFSS_AEDT_ROOT` path. Do not invent install paths.
3. Prefer no-GUI operation unless the user needs visual review or the workflow
   cannot expose the required state programmatically.
4. Before setup, solve, export, or result interpretation, inspect the active
   project/design using the best available path:

```bash
sim inspect session.versions
sim inspect session.summary
sim inspect hfss.project.identity
sim inspect hfss.design.summary
sim inspect hfss.model.summary
sim inspect hfss.boundaries.summary
sim inspect hfss.setups.summary
```

   Use release-specific API guidance only when `session.versions` reports
   `version_source: active_runtime`. A newer AEDT installation detected on the
   same host does not establish the version of the connected HFSS session.

   If you are not using `sim-cli`, collect equivalent project, design, object,
   boundary, port, setup, and sweep information through PyAEDT or AEDT APIs.
5. Run one bounded step at a time. After each mutation, inspect the changed
   state and capture `last.result`, script output, or equivalent logs.
6. Treat process success as transport success only. Engineering acceptance must
   come from HFSS results, exported data, convergence, S-parameters, fields,
   far-field quantities, or another domain-specific criterion requested by the
   user.
7. Keep failed evidence. If a solve/export fails, capture AEDT messages,
   stdout/stderr, generated files, and the exact step that failed.

## Stateful Parameter Optimization

Treat a TDR or S-parameter optimization as a recoverable state machine, not as
a chain of scripts in one chat. Before the first mutation, create a durable run
record beside the working project. Record the exact AEDT runtime and version,
source and working-copy identities, owned session PID, project/design/setup and
sweep/report context, objectives and constraints with units, parameter names and
bounds, the active point, and the paths and timestamps of result artifacts.
Update this record atomically after every state change so a restarted agent can
resume without reconstructing truth from conversation history.

Use these iteration-accounting rules consistently:

- A requested baseline that has not run is not yet an optimization iteration.
- A solve that is alive, or solved data that still needs a trustworthy numeric
  export, is pending. Do not count it or use it to choose another point.
- A point with an explicit solve, convergence, export, schema, or numeric
  validation failure is failed and must not count. Persist its parameters,
  messages, and artifact evidence; do not let it steer the optimizer.
- Count and commit a point only after the expected numeric rows were exported,
  units and columns were checked, the objective was computed, and every hard
  constraint was evaluated.
- A change to the allowed search space is metadata, not a result. If a solve is
  active, keep its parameters frozen, record the new scope for later, and do
  not restart or mutate the active point.

Only a committed baseline and committed candidate measurements may establish
an optimization direction. Compare the objective and constraints to the prior
committed point, keep the proposed value inside its recorded bounds, and change
one parameter per point unless the user explicitly selected a designed
multi-parameter study. Estimates and qualitative RF intuition may be presented
as estimates, but must not be promoted to validated HFSS evidence.

### Identity, Version, and Project Preservation

Never route automation from the active desktop window or a process name alone.
Discover sessions, then bind the intended one using ownership/PID plus the
expected project, design, and setup. Keep the full identity chain in the run
record and re-check it after reconnecting. Do not terminate, close, or modify a
different user-owned AEDT session.

Treat an AEDT executable path and an AEDT project path as distinct inputs. When
opening an older project in a newer runtime, preserve the source project,
create a separate migrated copy, launch the requested runtime, open only the
copy, inventory all designs, and verify the selected design/setup/report
context before any solve. A successful open proves neither model correctness
nor a numeric baseline.

If the user corrects an earlier availability assumption by confirming a valid
license and an already-open AEDT session, use that as environment evidence.
Discover and disambiguate the existing sessions; do not repeat installation or
license setup and do not attach to whichever window happens to be active.

For projects with multiple or similarly named designs, inventory the designs
before selecting a target. Verify the requested topology, layer context,
ports, setup, sweep, and report solution context. A matching name or active
design alone is insufficient.

### Baseline and Parameterization Gate

For a supplied project, first preserve a working copy, verify its identity and
current topology, define numeric objectives and constraints, map actual geometry
objects to bounded parameters, and save the parameterization plan. Establish a
numeric baseline in that working copy before proposing or starting candidate
points. Do not overwrite the original, silently guess object mappings, or jump
from a planning request into an optimization solve.

Accept a baseline only when the recorded runtime, working-copy project, design,
setup, and report solution context match, and the exported file has the expected
schema and numeric rows from which the baseline metric can be reproduced.
Historical plots or reports copied from another version are context, not the
migrated numeric baseline.

If there is no base project and the requested result must support engineering
review, ask for the missing stackup, conductor and via geometry, materials,
ports/excitations, boundary conditions, target topology, frequency/time setup,
and acceptance criteria. A prose request or scale-free image does not justify
inventing a review-ready model or claiming a result.

### Recovery Without Duplicate Solves

A control/API timeout is not a solver failure. On timeout, use the recorded
session identity to inspect the owned AEDT process, solver progress, solution
artifacts, and their modification times. If the process is alive and progress
or artifacts continue changing, leave the point pending, wait for the existing
solve, reconnect to that same owned session if necessary, then query/export and
validate the result. Never launch a duplicate solve merely because a tool call
returned or timed out.

Likewise, a report/export failure is not proof that the solve failed. If solved
data and fresh solution artifacts exist, inspect the report's setup/sweep/domain
binding, query the solution data directly, rebuild only the report binding when
needed, export again, and validate numeric rows before considering a rerun.

When the user completed a solve manually while the agent was waiting, adopt the
existing solved data as an unvalidated pending result. Verify the target
project/design/setup and solution/report context, inventory fresh artifacts,
query and export the data, validate it, and append it to the ledger. Do not
discard or duplicate a solve simply because the agent did not start it.

After an agent or application restart, load the durable run record before
choosing any action. Reconcile each recorded point with owned-process liveness,
solver progress, artifact presence/timestamps, report binding, and numeric
exports. Resume monitoring an alive progressing solve, recover an exportable
result, or persist a failed point; never start the next point until the current
state is reconciled.

### Claim Discipline

Keep conclusions no stronger than the evidence. Session readiness requires the
complete ownership/PID/project/design/setup chain. A running claim requires
owned-process liveness and progress evidence. A solved-data claim does not imply
a valid report or numeric result. A valid-iteration claim requires a checked
numeric export, objective, constraints, and persisted ledger entry. Report the
first missing gate explicitly instead of treating a script exit code, visible
plot, project-open event, or search-space update as engineering completion.

## Common Workflows

### Offline AEDT Project Probe

When the user only needs quick inventory or triage from a saved project and
AEDT may not be installed, use the experimental offline inspector before
launching AEDT:

```bash
python -m sim_plugin_hfss.aedt_inspect path/to/project.aedt
```

It can report best-effort project/design hints, variables, setup and sweep
names, port/boundary names, lock-file status, and `.aedtresults` sidecar
progress. It does not solve, read field data, extract mesh connectivity, or
provide authoritative geometry/boundary semantics. When accuracy matters,
validate the same file through PyAEDT/AEDT inspections.

### Connect Through sim-cli

```bash
sim check hfss
sim connect --solver hfss --ui-mode no_gui
sim inspect session.summary
sim inspect hfss.project.identity
sim inspect hfss.design.summary
```

Use GUI mode only when the user needs to watch or interact with AEDT:

```bash
sim connect --solver hfss --ui-mode gui
```

### Run a PyAEDT Script

Use this for a complete script that constructs or opens an HFSS project:

```bash
sim lint --solver hfss path/to/script.py
sim run --solver hfss path/to/script.py
```

Direct execution is also acceptable when it is clearer:

```bash
python path/to/script.py
```

In either case, preserve the AEDT project, logs, exported reports, and numeric
acceptance results.

### Execute a Bounded Snippet

After `sim connect`, snippets can use the live `hfss` object:

```python
{
    "project": hfss.project_name,
    "design": hfss.design_name,
    "setups": list(hfss.setup_names),
}
```

Prefer JSON-serializable results. Use `sim inspect last.result` after each
snippet.

Control-plane snippets are bounded by the HFSS driver by default. To tighten or
disable that bound for a session, pass an explicit driver option:

```bash
sim connect --solver hfss --ui-mode no_gui --driver-option exec_timeout_s=60
```

Use short bounds for inspection, setup edits, and exporter snippets. Do not use
a fixed wall-clock timeout as the failure signal for real solves; solve-like
snippets such as `analyze_setup(...)` are not given the driver's default control
timeout unless you explicitly set `exec_timeout_s`. If a snippet returns
`hung: true`, treat the session as quarantined: inspect `session.health`, then
reconnect before more HFSS work.

### Export and Parse S-Parameters

When Touchstone export is available, export `.sNp` and parse it directly for
acceptance metrics. The connector exposes a best-effort helper in snippets:

```python
touchstone_summary("path/to/result.s1p", target_frequencies_ghz=[5.8], threshold_db=-10)
```

Do not require `scikit-rf` just to compute minimum S-parameter, target-frequency
values, or threshold bandwidth.

### Long Solves, Sweeps, and Multi-Physics Coupling

There is no dedicated resumable-sweep primitive and no automatic HFSS-to-Icepak
coupling API, and none is planned. Drive both yourself with the generic
primitives above instead of waiting for a purpose-built one:

- **Multi-point sweeps**: run one point at a time through bounded `run()` /
  `sim exec` calls. Use `session.health` / `hfss.solution.progress` between
  points to tell "still converging" from "actually hung." A stuck point does
  not require restarting the whole sweep — the AEDT project retains completed
  points, so reconnect and continue from the next point.
- **HFSS-to-Icepak coupling**: there is no supported automatic EM-Loss link.
  Read per-object/per-frequency loss from HFSS through the same `run(code)`
  scripting used for inspection, then feed the computed value into Icepak's
  own (manual) heat-source assignment call. See
  `sim-cookbook/hfss/examples/cpw_thermoelectric_rf_power_sensor/` for the
  staged recipe (RF baseline -> manual heat source -> loss-driven heat source).

Ask for a new driver primitive only when a generic gap actually blocks
progress (for example, no timeout guard existed before v0.1.1) — not merely
because a specific workflow would be more convenient with a dedicated
shortcut.

## Monitoring and Evidence

Use these inspections when available:

- `hfss.model.summary` for object names, sheet/solid grouping, materials, and
  bounding boxes when PyAEDT exposes them.
- `hfss.boundaries.summary` for boundaries, excitations, ports, and associated
  objects when available.
- `hfss.setups.summary` for setup and sweep names/properties.
- `hfss.messages` for AEDT errors/warnings/info.
- `hfss.solution.progress` for best-effort solved frequency progress from
  `.aedtresults` files.
- `session.health` for PyAEDT/AEDT liveness, tracked owned AEDT PIDs, recent
  timeout cleanup, recent messages, and best-effort solve progress.

If an inspection is unavailable, report that honestly and use a solver-native
fallback. Do not fabricate status.

## Modeling Gotchas

- AEDT Student installs use the student launcher and may expose version strings
  such as `2025.2SV`. The connector patches PyAEDT 0.26.x startup handling for
  this case, but direct scripts may need the same care.
- If an `Electronics Desktop Student Help` dialog appears during automation,
  close it, record the exact command, and remove `-help` from the probe path.
  Use Registry/path/import checks for discovery, or run a real script
  invocation when the task needs AEDT execution.
- For small antenna feeds between curved conductors, do not rely on
  `lumped_port(..., create_port_sheet=True)` until the generated sheet has been
  visually or solver-message validated. HFSS can create a non-planar port sheet
  between round objects, which fails during port refinement. Prefer an explicit
  planar sheet plus an explicit two-point integration line.
- Save the project before a real solve smoke, and capture
  `hfss.odesktop.GetMessages(...)` immediately after `analyze_setup(...)`
  returns false. Reopening the project later can lose the most useful failure
  context.
- For GUI evidence on Windows, inspect the screenshot after taking it. If a
  full-desktop capture is black, capture the AEDT window by handle instead of
  treating the file's existence as visual proof.

## First-Version Limits

- Direct `.aedt` and `.aedtz` solving through `sim run` is not validated yet.
- Real HFSS release validation is opt-in and must be recorded separately from
  ordinary no-AEDT unit tests.
- Do not claim solver correctness from plugin unit tests alone.

## Troubleshooting

- Driver not discovered: only when using `sim-cli`, reinstall the plugin in
  the same environment as sim-cli and rerun `sim check hfss`.
- AEDT not detected: set `SIM_HFSS_AEDT_ROOT` to the directory containing an
  AEDT launcher, or rely on Registry/default discovery for common install
  layouts. A permanent global `PATH` change is optional, not required.
- PyAEDT import error: install `pyaedt>=0.26.3,<1` in the active environment.
- Script not detected: make sure it constructs HFSS through PyAEDT, for example
  `from ansys.aedt.core.hfss import Hfss` followed by `Hfss(...)`.
