# Multisector NUTS2 research: project roadmap

Working notes for studying the integrated OpenEMPIRE.jl Power / Natural Gas /
Hydrogen / CO2 (/ Industry) implementation and, later, investigating how it
could be combined with the NUTS2 Power model.

## Goal

1. Understand the existing Natural Gas, Hydrogen and CO2 modules: physics,
   mathematics, implementation and input data.
2. Only then investigate how they could work with the NUTS2 North Sea Power
   model (NUTS2 nodes, national nodes, offshore hubs).

Understanding comes first. No adaptation work is planned before Phase 8.

## Repositories and branches

| Purpose | Local path | Branch | Commit |
|---|---|---|---|
| Integrated implementation (this repo) | `OpenEMPIRE_Julia_Gas_Hydrogen` | `research/nuts2-multisector` | `cbf353d` (= upstream `main`) |
| Previous Power/NUTS2 work | `OpenEMPIRE_Julia_NUTS2_commit` | `feature/nuts2-power` | `5800578` |

Remotes in this repo: `origin` = personal fork
(`ParinazAliasghari/OpenEMPIRE.jl`), `upstream` = NTNU
(`ntnuiotenergy/OpenEMPIRE.jl`). `remote.pushDefault` is `origin`.

`feature/nuts2-power` and `research/nuts2-multisector` are both on the fork.
The NUTS2 repository holds large untracked datasets and results that are in no
commit; it is not touched during the study phases.

## Working rules

- Phases 1 to 7 are read-only study. No model runs, no tests, no conversion
  scripts, no dependency installs.
- Nothing is written to the NTNU repository. Every push names the fork URL
  explicitly.
- The two repositories are not merged at this stage.
- Disk space is limited: no extra clones, no duplicated datasets, no copied
  results.
- Claims are labelled **verified** (read in source or data), **interpretation**,
  or **unverified**. Code capability, existing data and future adaptation are
  kept apart: national-level data does not imply national-only code, and
  separate sector node sets do not imply mixed spatial resolution.

## Phases

| Phase | Topic | Status |
|---|---|---|
| 1 | Integrated architecture and spatial node structure | Done. See `01_integrated_architecture.md` |
| 2 | Natural Gas mathematical formulation | Parts A and B delivered (physical logic, equations). Parts C to G pending: input-data walkthrough, one real gas node, objective, spatial table, summary |
| 3 | Natural Gas input data | Pending |
| 4 | Hydrogen mathematical formulation | Pending |
| 5 | Hydrogen and CO2 input data | Pending |
| 6 | Cross-sector coupling | Pending |
| 7 | Existing tests and validation | Pending |
| 8+ | NUTS2 feasibility, mapping design, toy cases, controlled implementation | Not started |

## Standing open questions

Carried over from Phase 1 and Phase 2 (details in
`01_integrated_architecture.md`):

1. Hydrogen generators are not checked to sit at hydrogen nodes. Is that
   guarded anywhere else?
2. Transport electricity demand is added only at nodes that are both hydrogen
   production nodes and onshore. Does anything depend on that?
3. Gas-only nodes Draupner and Sleipner: how do they behave in the power
   balance?
4. Does every electricity arc have a `transmissionLength` row? Hydrogen and
   CO2 pipeline lengths depend on it.
5. Is the gas curtailment cost (labelled EUR/MWh) applied per ton as in
   InternalEMPIRE?
6. Storage "block" semantics (state resets per season/peak/scenario block):
   strongly supported by the TimeStruct source, not exhaustively traced.
7. The weather x gas scenario ordering and gas scenarios above 1 are blocked in
   validation (`empire_structs.jl:611`) and have no reference parity.
