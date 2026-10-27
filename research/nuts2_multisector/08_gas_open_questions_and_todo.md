# Natural Gas: open questions and to-do list (living document)

Commit `cbf353d3cd033cb0ebab1a062a0986e4560290b8` plus research notes. Update
this file as items are investigated. Do not mark anything "Confirmed" without
evidence (code, data or a run).

Types: **Question** (behaviour unclear), **Suspected issue** (possible bug or
inconsistency, not confirmed), **Improvement** (future idea), **NUTS2**
(spatial adaptation consideration; observation only, no design).
Status: **Open**, **In progress**, **Resolved**, **Won't pursue**.
Paths are relative to `src/` unless they start with `data/`. Series docs: 03 to 07.

## 1. Open items

| ID | Type | Topic | Status | Notes / Code Reference |
|---|---|---|---|---|
| G-01 | Suspected issue | Gas-only output writer may index transport variables that do not exist | Open | `results.jl:1255-1268` indexes `transport_met[node,t]` for every onshore node; in a YAML gas-only run these are never inserted (`natural_gas.jl:156-161`, `user_interface.jl:292`). Not run; behaviour of the sparse container on a missing index not confirmed. Tests use the lower-level API (`test/test_natural_gas.jl:79, 888`). |
| G-02 | Suspected issue | Transport-demand activation differs between entry points | Open | `create_model` passes `hydrogen_enabled` (`user_interface.jl:292`); `create_variables`/`create_constraints` default to `natural_gas` (`model_definition.jl:86, 220`). Same gas-only setup gives different models by path. Intent unknown. |
| G-03 | Question | Curtailment cost units | Open | `Transport/CurtailCost.csv` header `CurtailCost_(€/MWh)`, value 100000, applied to a ton/h shed variable (`natural_gas.jl:437-445`). Same constant also prices Hydrogen shedding (`hydrogen.jl:890-898`). Compare with InternalEMPIRE. |
| G-04 | Question | Italy reserve applied to two terminals | Open | Italy has DomesticProduction and PipelineImport but one value in `Reserves.csv`; the row is built per (node, terminal) (`natural_gas.jl:192-236`). Italy source duplicate resolved to 28,830,371.65 (`NaturalGas/reserves_duplicate_audit.csv`). Is per-terminal use intended? |
| G-05 | Question | Single-scenario behaviour of `opscenarios` | Open | Reserve rows pick scenario `(w-1)*G+gs` via `_operational_scenario_at` (`natural_gas.jl:199-205`) and storage uses `opscenarios(strategic_period)`. With one scenario `create_timestruct` uses plain `SimpleTimes` (`model_definition.jl:47`). Not traced. |
| G-06 | Question | Role of Sleipner and Draupner in the power balance | Open | Gas nodes without terminals; each has a `GasOCGT`, electricity links only to NO2/NO5 (per Phase 2 data check), and gas-only arcs to neighbours. How gas arrives and how the power balance treats them not traced. |
| G-07 | Question | 14 gas arcs with zero capacity | Open | `NaturalGas/PipelineCapacity.csv`, e.g. `Austria,CzechR,0.0`, `Hungary→Austria 0.0`. Closed routes kept for completeness, or data gaps? |
| G-08 | Question | Hard-coded 8760 in transport demand | Open | `natural_gas.jl:364` instead of the model's `operational_hours_per_year`. Check parity with InternalEMPIRE. |
| G-09 | Question | Hard-coded energy content 13.9 MWh/ton and storage constants | Open | `empire_structs.jl:24-27`; used by Gas, Hydrogen, Industry. Source of 13.9 and 50 % / efficiency 1.0 not documented in the repo. |
| G-10 | Question | Storage semantics for long-term use | Open | Inventory restarts at 50 % every (representative period, scenario) block, so no seasonal carry-over (`natural_gas.jl:297-326`; `TimeStruct/utils.jl:22-26`). Confirm this matches the intended physical meaning and the reference model. |
| G-11 | Suspected issue | Storage variables created at nodes with zero capacity | Open | 37 gas nodes get storage variables, 17 have capacity (`data/.../StorageCapacity.csv`); other nodes are forced to 0. Extra columns only; effect on solve time not measured. |
| G-12 | Question | Transport demand and Hydrogen flag | Open | Why gas transport demand exists only with Hydrogen (comment at `natural_gas.jl:154-155` says InternalEMPIRE declares it in the Hydrogen block). Gas-only runs read and validate the file but do not use it. |
| G-13 | Question | `TerminalCost_stochastic.csv` has one scenario | Open | 308 rows, gas scenario 1 only; validation forces G = 1 (`empire_structs.jl:611`). Gas scenarios above 1 and weather x gas ordering have no reference parity. |
| G-14 | Question | Can gas-only arcs be repurposed to hydrogen? | Open | Hydrogen corridors derive from electricity arcs (`read_csv.jl:256-262`); the 7 gas-only pairs touch Sleipner/Draupner. Only read from code, not confirmed. |
| G-15 | Question | `OnshoreNode` not validated against gas nodes | Open | Only required to be in `Node` (`empire_sets.jl:630`); Hydrogen/CO2 use it directly (`hydrogen.jl:153, 521`). Is a mismatch possible in practice? |
| G-16 | Question | Where YAML `leap_years_investment` becomes `duration_strat` | Open | `TwoLevel(npers, years_period, ...)` at `model_definition.jl:70`; used in reserve rows and discounting. Mapping from config not traced. |
| G-17 | Question | Gas generator recognition by name | Open | "gas" substring (`read_csv.jl:651`); Industry plants by "ng" (`industry.jl:319-344`). Could a new technology name match by accident? |
| G-18 | Improvement | Report more gas diagnostics | Open | Duals of terminal-capacity, reserve (`emp[:natural_gas_max_reserves]`), pipeline-capacity rows and cumulative reserve use are not written. |
| G-19 | NUTS2 | Spatial key is node-name equality | Open | Gas, power, Hydrogen and CO2 share names only; gas generators must be at a gas node (`empire_sets.jl:646-655`). Observation for later. |
| G-20 | NUTS2 | Totals attached to nodes | Open | Reserves, storage, terminal capacity and transport demand are totals per node (`NaturalGas/*.csv`, `Transport/NaturalGasDemand.csv`); no allocation rule exists for finer nodes. |
| G-21 | NUTS2 | Gas pipelines have no lengths | Open | No length data in the gas module; Hydrogen/CO2 take lengths from electricity lines (`hydrogen.jl:203-210`). Does every electricity arc have a `transmissionLength` row? |
| G-22 | NUTS2 | Period keys are positions | Open | Gas CSVs use period index 1..7, not years; changing the period structure changes all gas tables. |

## 2. Resolved items

| ID | Topic | Conclusion |
|---|---|---|
| R-01 | Does gas storage exist? | Yes: three variables per node and three constraints (balance, cyclic, maximum), capacity from `StorageCapacity.csv` (`natural_gas.jl:297-335`). |
| R-02 | Meaning of `objective_weight` | Defined in TimeStruct 0.9.6: probability x discount x multiple; with `avg_year` the period duration is inside the discount sum (`TimeStruct/src/discount.jl:167-204`). |
| R-03 | Where gas adjacency is built | In the `NaturalGasSets` constructor (`empire_sets.jl:188-197`). |
| R-04 | Storage reset behaviour | Inventory restarts at 50 % at the first hour of each (representative period, scenario) block; one cyclic row per block (`TimeStruct/utils.jl:22-26`; `op_scenarios/strat_periods.jl:226-230`). |
| R-05 | Is `industry.jl:608` a gas link? | No: it is a biomass constraint that loops over gas nodes. |
| R-06 | Are objective components reported? | Yes in the run manifest and console (`scripts/run_julia_empire.jl:759-764, 826-852`); `results_objective.csv` has only the total. |
| R-07 | Hydrogen storage node count | 36 nodes, 49 (node, storage) pairs, 13 nodes with two types (corrected from an earlier miscount). |
| R-08 | Gas pipeline lengths | None in the gas module. |
| R-09 | Is the gas module independent of Hydrogen? | Gas flows and generators yes; transport demand, reformer gas and repurposing need Hydrogen (see G-02, G-12). |

## 3. How to use this file

- Add new items with the next free ID (G-23, ...).
- When an item is resolved, move it to section 2 with a one-line conclusion and
  evidence (file and line, or run output).
- Keep suspected issues in section 1 until reproduced or ruled out.
