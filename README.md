# cfdstack

CFD simulation skills for Claude Code. Turns Claude into a computational fluid dynamics assistant — scaffold cases, validate meshes, run solvers, post-process results, debug convergence, validate against benchmarks, and generate reports. All slash commands, all Markdown, no compiled binaries required.

## Skills

| Skill | Phase | What it does |
|-------|-------|-------------|
| `/cfd-new` | **Setup** | Scaffold a new CFD simulation case. Asks what solver, physics, and geometry — generates the full directory structure with boundary conditions and solver settings. |
| `/cfd-check` | **Validate** | Check mesh quality and case setup before running. Runs `checkMesh`, validates boundary conditions, checks field dimensions, flags configuration errors. |
| `/cfd-run` | **Solve** | Run the CFD solver and monitor convergence. Selects the right solver, supports parallel execution, watches residuals, alerts on divergence. |
| `/cfd-post` | **Post-process** | Extract results and generate visualizations. Force coefficients, residual plots, velocity profiles, ParaView and matplotlib scripts. |
| `/cfd-debug` | **Debug** | Diagnose convergence and stability problems. Identifies root cause before suggesting any fix. CFL analysis, NaN tracing, BC validation, scheme recommendations. |
| `/cfd-validate` | **Validate** | Compare results against reference data or analytical solutions. Error norms, benchmark tables, grid convergence index (GCI). |
| `/cfd-report` | **Report** | Generate a professional simulation report in Markdown (or PDF with pandoc). Case setup, mesh stats, convergence, results, validation. |

## Quick start

```
You:    I want to simulate flow over a cylinder at Re=100.
You:    /cfd-new
Claude: [asks solver, physics, Re number, mesh resolution]
        [generates OpenFOAM case: 0/, constant/, system/]
        [sets BCs, fvSchemes, fvSolution, blockMeshDict]

You:    /cfd-check
Claude: [runs checkMesh, validates BCs, checks field dimensions]
        [PASS: mesh OK, all patches matched]

You:    /cfd-run
Claude: [runs icoFoam, monitors residuals]
        [CONVERGED at t=10s, Ux < 1e-5, p < 1e-4]

You:    /cfd-post
Claude: [extracts Cd, Cl vs time]
        [generates residuals.png, force_coefficients.png]

You:    /cfd-validate
Claude: [Cd=1.35, ref=1.33±0.05: PASS (1.5% error)]

You:    /cfd-report
Claude: [generates simulation_report.md]
```

## Install — 30 seconds

**Requirements:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Git](https://git-scm.com/)

### Step 1: Install globally (for your user account)

Open Claude Code and paste this:

> Install cfdstack: run **`git clone https://github.com/younglin90/cfdstack.git ~/.claude/skills/cfdstack && cd ~/.claude/skills/cfdstack && ./setup`** then add a "cfdstack" section to CLAUDE.md listing the available skills: /cfd-new, /cfd-check, /cfd-run, /cfd-post, /cfd-debug, /cfd-validate, /cfd-report.

### Step 2: Add to your project so teammates get it (optional)

> Add cfdstack to this project: run **`cp -Rf ~/.claude/skills/cfdstack .claude/skills/cfdstack && rm -rf .claude/skills/cfdstack/.git && cd .claude/skills/cfdstack && ./setup`** then add a "cfdstack" section to this project's CLAUDE.md listing the skills: /cfd-new, /cfd-check, /cfd-run, /cfd-post, /cfd-debug, /cfd-validate, /cfd-report. Tell Claude that if skills aren't working, run `cd .claude/skills/cfdstack && ./setup`.

### Or run manually

```bash
git clone https://github.com/younglin90/cfdstack.git ~/.claude/skills/cfdstack
cd ~/.claude/skills/cfdstack && ./setup
```

Then add to your project's `CLAUDE.md`:

```
## cfdstack
Available skills: /cfd-new, /cfd-check, /cfd-run, /cfd-post, /cfd-debug, /cfd-validate, /cfd-report
```

## Supported solvers

- **OpenFOAM** (primary) — icoFoam, simpleFoam, pimpleFoam, rhoCentralFoam, buoyantSimpleFoam, interFoam, and more
- **SU2** — compressible/incompressible RANS (partial support)
- **Custom solvers** — skills read your project's `CLAUDE.md` and adapt

## The CFD sprint

cfdstack follows the natural CFD workflow:

```
Think → Setup → Check → Run → Post → Validate → Report
        /cfd-new  /cfd-check  /cfd-run  /cfd-post  /cfd-validate  /cfd-report
                                              ↑
                                         /cfd-debug (when things go wrong)
```

Each skill feeds into the next. `/cfd-new` generates a `.cfdstack` config file that
downstream skills read. `/cfd-check` catches errors before you waste compute time.
`/cfd-debug` traces root causes before touching any settings.

## Troubleshooting

**Skill not showing up?**
```bash
cd ~/.claude/skills/cfdstack && ./setup
```

**OpenFOAM not found?**
Make sure OpenFOAM is sourced before running Claude Code:
```bash
source /opt/openfoam*/etc/bashrc   # Linux
source ~/OpenFOAM/*/etc/bashrc     # macOS/local install
```
Or add it to your shell's `.bashrc`/`.zshrc`.

**Claude says it can't see the skills?** Add this to your project's `CLAUDE.md`:
```
## cfdstack
Available skills: /cfd-new, /cfd-check, /cfd-run, /cfd-post, /cfd-debug, /cfd-validate, /cfd-report
```

## License

MIT. Free forever.
