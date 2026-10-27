# Integrated architecture and spatial node structure (Phase 1)

Based on commit `cbf353d3cd033cb0ebab1a062a0986e4560290b8`. Line numbers are
relative to `src/` unless a path is given. Findings were read from source and
from `data/full_model_int`; nothing was run.

## 1. One model, one objective

| Question | Answer | Reference |
|---|---|---|
| Entry point | runner `main()` -> `_run_model` -> `OpenEMPIRE.create_model` | `scripts/run_julia_empire.jl:1126, 947`; `user_interface.jl:218` |
| JuMP model created | `JuMP.Model()` or `JuMP.direct_model(optimizer)` | `user_interface.jl:279-283` |
| Activation | YAML booleans `natural_gas`, `hydrogen`, `industry` (default `false`) | `scenario.jl:82-84` |
| Order | power -> gas -> hydrogen (+CO2) -> industry, for both variables and constraints | `model_definition.jl:178-182`, `:245-304` |
| Same model object | yes; modules read each other's variables via `emp[:name]` | e.g. `natural_gas.jl:269`, `:386-391` |
| Objective | `@objective(emp, Min, sum(values(components)))`, 17 named components | `model_definition.jl:202`, `:815-932` |
| `optimize!` | `JuMP.optimize!(emp)` | `scripts/run_julia_empire.jl:816` |
| CO2 | not a separate module: built inside the Hydrogen functions, loaded only with `hydrogen: true` | `hydrogen.jl:75`, `:269` |

Dependency gates (`user_interface.jl:129-140`): `hydrogen` and `industry` both
require `natural_gas`; both require `number_of_gas_scenarios = 1`. Gas
validation also rejects `gasScenarioCount != 1` for gas-only runs
(`empire_structs.jl:611`). The three modules need the CSV dataset layout
(`read_csv.jl:95, 135`).

```
runner main -> _run_model -> create_model
   _prepare_model_inputs -> read_data -> read_sets_csv -> validate!(sets)
   preprocess_params -> validate / validate_natural_gas / validate_hydrogen
   emp = JuMP.Model()
   create_variables      power -> gas -> hydrogen+CO2 -> industry
   create_constraints    power flow_balance -> gen/stor/trans/emission
                         -> gas -> hydrogen+CO2 -> industry
   create_objective      one Min over 17 components
   optimize!(emp) -> write_solution_tables
```

## 2. Spatial sets

There is one universal node set, `Node`, plus sector subsets. Each subset has
its own CSV, and `validate!` (`empire_sets.jl:524-790`) enforces the subset
relations.

| Sector | Julia set | Source CSV | Relation |
|---|---|---|---|
| Power | `sets.Node` / `nodes(sets)` | `Sets/Node.csv` (52) | universe |
| Power | `sets.DirectionalLink` | `Sets/DirectionalLink.csv` (436 arcs) | ends in `Node` |
| Power | `OffshoreWindFarmNode`, `OffshoreEnergyHub` | `Sets/OffshoreWindFarmNode.csv` (14), `OffshoreEnergyHub.csv` (1) | subsets of `Node`, disjoint |
| Gas | `NaturalGas.Node` | `Sets/NaturalGasNodes.csv` (37) | subset of `Node` (:628) |
| Gas | `NaturalGas.DirectionalLink` | `Sets/NaturalGasDirectionalLines.csv` (94 arcs) | own arc list, not tied to electricity arcs |
| Gas | `NaturalGas.OnshoreNode` | `Sets/OnshoreNode.csv` (35) | used by gas, hydrogen and CO2; subset of `Node` (:630) |
| Gas | `NaturalGas.Generator` | none: generator name contains "gas" (`read_csv.jl:651`) | must sit at a gas node (:650-655) |
| Hydrogen | `Hydrogen.ProductionNode` | `Hydrogen/ProductionNodes.csv` (36) | subset of `Node` (:674); must contain all onshore nodes (:706) |
| Hydrogen | `ReformerLocation`, `TerminalNode`, `StoragesOfNode` | `ReformerLocations.csv` (35), `H2TerminalNodes.csv` (15), `StorageMaxCapacity.csv` (36 nodes, 49 node-storage pairs) | reformers in gas nodes (:709); terminals and storage in production nodes (:713, :699-704) |
| Hydrogen | `Hydrogen.DirectionalLink` | derived from electricity arcs with both ends in production nodes (`read_csv.jl:256-258`) | filtered copy of electricity arcs |
| Hydrogen | `Hydrogen.Generator` | `Hydrogen/HydrogenGenerators.csv`; must equal "name contains hydrogen" (`read_csv.jl:238-242`) | no node check found |
| CO2 | `CO2SequestrationNode` | `CO2/CO2SequestrationNodes.csv` (6) | subset of onshore (:711) |
| CO2 | `CO2DirectionalLink` | derived from electricity arcs with both ends onshore (`read_csv.jl:264`) | filtered copy of electricity arcs |
| Industry | Steel/Cement/Ammonia/Oil producers | `Sets/*Producers.csv` | subset of gas nodes (:755), of hydrogen nodes if refinery active (:759) |

Hydrogen and CO2 pipeline lengths come from the electricity line lengths
(`hydrogen.jl:203-210`, `Transmission/transmissionLength.csv`).

### Requirement versus dataset convention

- Requirement (checked in `validate!`): every sector set is a subset of
  `Node`; a gas generator is located at a gas node; onshore nodes are hydrogen
  production nodes when hydrogen is on; reformers are at gas nodes; CO2 sites
  are onshore.
- Dataset convention only: gas nodes = onshore + {Sleipner, Draupner};
  hydrogen nodes = onshore + {EnergyhubEU}.
- No explicit mapping set exists between sector nodes.

## 3. Resolution of `data/full_model_int`

National nodes (country names, Norway as NO1 to NO5), 14 offshore wind-farm
zones, one offshore hub (`EnergyhubEU`) and two gas-platform nodes (Sleipner,
Draupner, judged from names only, unverified). Resolution is implied by names;
no data file encodes it, and "NUTS" does not appear in the repository. All
sectors reuse the same node strings. No sector is finer or coarser than another;
sectors differ in coverage only.

Code capability is not national-only: the code only uses node strings. Whether
it supports a sector node that aggregates several power nodes is a separate
question; no mechanism for that was found.

## 4. Cross-sector connections (all by node-name equality)

| Link | Variables | Constraint | Lines |
|---|---|---|---|
| Gas -> gas generation | `ngForPower[node, gen, t]` and `genOperational[node, gen, t]` | `natural_gas_for_power`, `natural_gas_flow_balance` | `natural_gas.jl:23-27, 146-148, 271-280, 366-403` |
| Power -> electrolysis | `electrolyzerElectricity[node, t]` -> `electrolyzerHydrogen[node, t]` | `hydrogen_electrolyzer_conversion`, power `flow_balance`, `hydrogen_flow_balance` | `hydrogen.jl:140-141, 212-223, 301-307, 454-457`; `model_definition.jl:245-259` |
| Hydrogen -> hydrogen generation | `hydrogenForPower[node, gen, t]` and `genOperational[node, gen, t]` | `hydrogen_for_power`, `hydrogen_flow_balance` | `hydrogen.jl:1-5, 189-191, 290-300, 479-483` |

Generators are identified by name substrings ("gas", "hydrogen") and located
through the power `GeneratorsOfNode` pairs. Electrolyzers have no per-node
input; every hydrogen production node can host one.

## 5. Real example

- Germany: node in power, gas, hydrogen, reformer and H2-terminal sets; not a
  CO2 site. Generators include `GasCCGT` and `HydrogenCCGT`; gas terminals
  RussianGas, LNGImport, DomesticProduction.
- NO2: also a CO2 sequestration site (`MaxSequestrationCapacity` 2.95e10 t) and
  a DomesticProduction gas terminal.
- There is no national "Norway" node; Norway appears as NO1 to NO5.

## 6. Preliminary reading for NUTS2 (conceptual, not a plan)

Hypothetical: power nodes NO1, NO2, NO3 and gas/hydrogen node `Norway`.

- Gas generator at NO2 fed by gas node `Norway`: not supported directly. The
  generator's node must itself be a gas node (`empire_sets.jl:650-655`), and
  the gas balance only sees generators located at the gas node of the same name.
- Electrolyzer at NO1 injecting into hydrogen node `Norway`: not supported
  directly. Electrolyzer electricity is drawn at the power node with the same
  name as the hydrogen node (`hydrogen.jl:221-223`).

Architectural assumptions to keep in mind later: node-name equality for
coupling; sector sets as subsets of `Node`; hydrogen and CO2 arcs derived from
electricity arcs with electricity line lengths; `OnshoreNode` stored in the gas
sets but used by hydrogen and CO2.

## 7. Open questions

See the standing list in `00_project_roadmap.md`.
