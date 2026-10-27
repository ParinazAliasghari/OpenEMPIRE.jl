# Natural Gas documentation 1/5: sets and CSV data dictionary

Part of the Natural Gas documentation series (03 to 07). Source of truth: the
OpenEMPIRE.jl code at commit `cbf353d3cd033cb0ebab1a062a0986e4560290b8`, with
data from `data/full_model_int`. Paths and line numbers are relative to `src/`
unless they start with `data/`. Nothing was run.

Labels: **[V]** verified in source or data, **[I]** interpretation,
**[U]** unverified or open.

Series: 03 sets and CSV data (this file) · 04 parameters and lineage ·
05 decision variables · 06 constraints · 07 objective, dependencies, outputs and
worked examples.

---

## 1. Activation and file layout

- The module is active only when the YAML key `natural_gas` is `true`
  (`scenario.jl:82`, default `false`). [V]
- It needs the CSV dataset layout (`read_csv.jl`), and requires
  `number_of_gas_scenarios = 1` (`empire_structs.jl:611`). [V]
- Sets are read by `read_sets_csv` -> `_read_natural_gas_sets_csv`
  (`read_csv.jl:626-661`, called at `:678`). Numeric parameters are read by
  `_read_natural_gas_params_csv` (`read_csv.jl:735-914`, called from
  `read_params_csv`, `:1507`). [V]
- Common reader rules (`_required_csv`, `_validated_module_rows`,
  `_insert_unique_module_value!`, `_module_nonnegative`; `read_csv.jl:344-468`,
  `:726`): exact file-name case, exact header, finite non-negative numbers,
  positive integer periods, and **duplicate keys throw**. [V]

## 2. Set dictionary

`NaturalGasSets` (`empire_sets.jl:69-91`, keyword constructor `:169-209`).
Adjacency lookups are built once in the constructor (`:188-197`). [V]

| Set | Julia field | Source | Elements (full_model_int) | Membership rules | Main consumers |
|---|---|---|---|---|---|
| Gas nodes | `Node` | `Sets/NaturalGasNodes.csv` (`NaturalGasNodes`) | 37 | unique, non-empty, all in power `Node` (`empire_sets.jl:624-628`) | every gas variable and constraint |
| Gas arcs | `DirectionalLink` | `Sets/NaturalGasDirectionalLines.csv` (`NodeFrom,NodeTo`) | 94 directed arcs | unique, ends in gas nodes, no self-loops (`:632-637`) | `ngTransmission`, capacity, balance |
| Terminal types | `Terminal` | `Sets/NaturalGasTerminals.csv` | 4 (DomesticProduction, LNGImport, PipelineImport, RussianGas) | unique, non-empty | `ngTerminalImport` index |
| Terminals of node | `TerminalsOfNode` | `Sets/NaturalGasTerminalsOfNode.csv` (`Node,NG_Terminal_Type`) | 44 pairs at 28 nodes | node in gas nodes, type in terminal types (`:638-645`) | supply variables and limits |
| Onshore nodes | `OnshoreNode` | `Sets/OnshoreNode.csv` | 35 | all in power `Node`; **not required to be a subset of gas nodes** (`:630`) | transport demand; also Hydrogen and CO2 |
| Gas generators | `Generator` | derived: generator name contains "gas" (case-insensitive) | 5 (Gasexisting, GasOCGT, GasCCGT, GasCCSadv, GasCCS) | all in `Generator`; every (node, generator) pair of a gas generator must be at a gas node (`:646-655`) | `ngForPower`, `natural_gas_for_power` |
| Incoming / Outgoing | `Incoming`, `Outgoing` | derived from `DirectionalLink` | per node | none | flow balance, pipeline electricity |
| Terminals by node | `TerminalsByNode` | derived from `TerminalsOfNode` | per node | none | flow balance |

Accessors: `natural_gas_nodes`, `_links`, `_terminals`, `_terminal_nodes`,
`_onshore_nodes`, `_generators`, `_incoming`, `_outgoing`
(`empire_sets.jl:490-503`). `has_natural_gas` is true when the node set is
non-empty. Two filtered helpers live in `natural_gas.jl`:
`_natural_gas_onshore_nodes` (onshore ∩ gas nodes, `:20`) and
`_natural_gas_node_generators` (generator pairs at gas nodes, `:23`). [V]

### Relation to the power node set

- Gas nodes = 35 onshore nodes + `Sleipner` + `Draupner`, all inside the 52
  power nodes. The 15 power nodes not in the gas set are the 14 offshore
  wind-farm nodes and `EnergyhubEU`. [V]
- The link between sectors is name equality; there is no mapping set. [V]
- 28 gas nodes have a terminal; 9 do not (CzechR, Draupner, Luxemb., NO1, NO4,
  Sleipner, Slovenia, Sweden, Switzerland). [V]
- 148 (node, gas generator) pairs sit at 36 gas nodes; only `BosniaH` has none.
  [V]
- 86 of the 94 gas arcs are also electricity arcs. The 7 undirected pairs that
  are not (Belgium-Sleipner, GreatBrit.-Sleipner, NO3-Sleipner, Draupner-France,
  Draupner-Germany, Draupner-NO5, Draupner-Sleipner) all touch the two platform
  nodes. [V]

### Index conventions used by the gas data

- **Period** in every gas CSV is the **position of the strategic period**
  (1..7). `Sets/Period.csv` has 7 entries and is not a year. In code the
  context map gives `period_context[t].strategic` (`natural_gas.jl:29-49`). [V]
- Gas scenario is always 1 in the shipped data. [V]
- Weather and gas scenario of an operational scenario are
  `weather = fld(c-1, G)+1`, `gas = mod(c-1, G)+1` (`scenario.jl:92-95`). [V]

## 3. CSV data dictionary

Rows counted from `data/full_model_int`. "Defined in" is the reader;
"Validated in" is `_check_natural_gas_params!` (`empire_structs.jl:572-707`)
unless stated otherwise.

### 3.1 Sets files

| File | Header | Rows | Sample | Reader |
|---|---|---|---|---|
| `Sets/NaturalGasNodes.csv` | `NaturalGasNodes` | 37 | `Austria` | `read_csv.jl:631` |
| `Sets/NaturalGasDirectionalLines.csv` | `NodeFrom,NodeTo` | 94 | `CzechR,Austria` | `:635` |
| `Sets/NaturalGasTerminals.csv` | `NaturalGasTerminals` | 4 | `DomesticProduction` | `:639` |
| `Sets/NaturalGasTerminalsOfNode.csv` | `Node,NG_Terminal_Type` | 44 | `NO2,DomesticProduction` | `:643` |
| `Sets/OnshoreNode.csv` | `OnshoreNode` | 35 | `Austria` | `:647` |

Terminal-type counts among the 44 pairs: DomesticProduction 14, LNGImport 16,
RussianGas 12, PipelineImport 2 (Spain, Italy). [V]

### 3.2 Parameter files

| File | Columns (unit) | Rows | Sample | Parameter | Reader | Missing or duplicate behaviour |
|---|---|---|---|---|---|---|
| `NaturalGas/PipelineCapacity.csv` | `FromNode, ToNode, Capacity_(ton/h)` | 94 | `Austria,CzechR,0.0` | `pipelineCapacity[(from,to)]` | `:750-773` | each gas arc must have a row, and each row must be a gas arc (`:639-641, :677`) |
| `NaturalGas/PipelineElectricityUse.csv` | `Power_usage_[MWh/ton]` | exactly 1 | `0.02444604019447059` | `pipelinePowerDemandPerTon` | `:775-785` | other row counts throw |
| `NaturalGas/StorageCapacity.csv` | `Node, Storage_(ton)` | 17 | `Germany,17647482.01` | `storageCapacity[node]` | `:787-798` | keys must be gas nodes; **missing nodes default to 0** |
| `NaturalGas/Reserves.csv` | `Node, Reserves_(tons)` | 15 | `Germany,13368462.36` | `reserves[node]` | `:787-798` | required for every node with a finite-reserve terminal (`empire_structs.jl:690-694`) |
| `NaturalGas/TerminalCapacity.csv` | `Node, Terminal, Period, Capacity_(ton/hr)` | 308 = 44 pairs x 7 | `Germany,DomesticProduction,1,481.625` | `terminalCapacity[(n,k,p)]` | `:800-829` | all pairs x all periods required |
| `NaturalGas/TerminalCost.csv` | `Node, Terminal, Period, Scenario, Cost_(EUR/ton)` | 308 | `Germany,DomesticProduction,1,1,278.0` | `terminalCost[(n,k,p,s)]` | `:831-870` | all pairs x periods x gas scenarios required; accessor default 99999 is never reached after validation |
| `NaturalGas/TerminalCost_stochastic.csv` | same, header `GasScenario` | 308 (scenario 1 only) | | same field | `:831` | read only when `number_of_gas_scenarios != 1`, which validation rejects |
| `Transport/NaturalGasDemand.csv` | `Node, Period, Natural_gas_demand_[MWh/yr]` | 245 = 35 x 7 | `Germany,1,506030485.89` | `transportDemand[(node,period)]` | `:872-900` | must cover **every** onshore node x period, even in gas-only runs |
| `Transport/CurtailCost.csv` | `CurtailCost_(€/MWh)` | exactly 1 | `100000` | `transportCurtailCost` | `:902-912` | other row counts throw |

The reader accepts `GasScenario` or `Scenario` as the fourth `TerminalCost`
header (`:836-839`). [V]

### 3.3 Audit files (not read by Julia)

`NaturalGas/reserves_duplicate_audit.csv` (1 row) and
`NaturalGas/terminal_cost_duplicate_audit.csv` (70 rows) record duplicate source
rows resolved by the dataset converter (last row wins, to mimic the Pyomo
DataPortal; see `docs/natural_gas_terminal_cost_duplicates.md`). The Julia
reader never sees duplicates. Example: Italy reserve 2,181,585,300 (discarded,
source row 7) vs 28,830,371.65 (kept, source row 11). [V]

### 3.4 Files from other modules that the gas module consumes

| File | Columns | Used for |
|---|---|---|
| `Sets/Generator.csv`, `Sets/GeneratorsOfNode.csv` (`Node,Generator`, 148 gas pairs) | | which gas generators exist and where |
| `Generator/genEfficiency.csv` | `GeneratorTechnology, Period, generatorEfficiency` (GasCCGT 0.574 in period 1) | gas-to-power conversion; required for each gas generator |
| `Generator/genFuelCost.csv` | 35 Gas rows | **ignored** for gas generators when gas is on (`utils.jl:268-273`) |
| `Generator/genVariableOMCost.csv`, `genCO2TypeFactor.csv`, `genCO2Captured.csv` | | marginal cost, emissions, CCS capture (see doc 07) |

### 3.5 Quantities not in any CSV

`mwhPerTon = 13.9`, `storageInitialFraction = 0.5`, storage charge and
discharge efficiency `1.0`, the hours-per-year constant `8760` in the transport
constraint, and the row-scale `1e-3` are fixed in code
(`empire_structs.jl:23-27`; `natural_gas.jl:11, 364`). [V]

## 4. Spatial and NUTS2 notes for this document (observations only)

- Every key is a node-name string. Gas keys have no relation to power keys
  except equality of the string. [V]
- Pipelines have capacities but **no lengths** in the gas module. Hydrogen and
  CO2 pipeline lengths come from the electricity line file. [V]
- `OnshoreNode` is shared by the gas, Hydrogen and CO2 modules and is not
  validated against the gas node set. [V]
- Generators are recognised by name substring ("gas"). A future technology
  name containing that text would automatically become a gas generator and
  need a gas-node location. [I from `read_csv.jl:651`]
- Period keys are positions, so changing the strategic-period count changes
  every gas table. [V]

## 5. Open questions for this document

1. Why the platform nodes `Sleipner` and `Draupner` have electricity links but
   no terminals, and how the power balance treats them. [U]
2. Why 14 pipeline arcs have capacity 0. [U]
3. `TerminalCost_stochastic.csv` contains one scenario only. Whether a real
   two-scenario file was ever intended. [U]
