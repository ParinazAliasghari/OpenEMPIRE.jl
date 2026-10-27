# Natural Gas documentation 5/5: objective, cross-sector dependencies, outputs and worked examples

Part 5 of the series. Commit `cbf353d3cd033cb0ebab1a062a0986e4560290b8`, data
`data/full_model_int`. Paths and line numbers are relative to `src/` unless they
start with `data/` or `scripts/`. Nothing was run.

Labels: **[V]** verified, **[I]** interpretation, **[U]** unverified.

---

## 1. Objective function

### 1.1 Structure

There is a single `@objective(emp, Min, sum(values(components)))`
(`model_definition.jl:202`), built from the named tuple returned by
`objective_component_expressions` (`model_definition.jl:815-932`). Gas adds two
components, computed in `natural_gas_objective_expressions`
(`natural_gas.jl:405-446`). [V]

| Component | Expression | Where |
|---|---|---|
| `natural_gas_terminal_import` | $\displaystyle\sum_{(n,k)}\sum_{t} w_t\;c_{n,k,p(t),s(t)}\;x_{n,k,t}$ | `natural_gas.jl:425-436`; collected at `model_definition.jl:920` |
| `natural_gas_transport_shedding` | $\displaystyle\sum_{n\in\text{onshore}}\sum_{t} w_t\;\kappa\;h_{n,t}$ | `natural_gas.jl:437-445`; collected at `model_definition.jl:921` |

Both sums use `objective_weight(t, discounter; type = "avg_year")`, and both are
accumulated in place with `add_to_expression!` to avoid quadratic expression
growth (`natural_gas.jl:420-424`). [V]

No other gas cost exists: pipelines, storage, gas transport losses and
investments carry no cost, and gas generators have **zero fuel cost** in their
marginal cost (`utils.jl:268-273`). The remaining cost of a gas generator
(variable O&M and carbon) stays in the power component
`generator_operation`. [V]

### 1.2 The weight $w_t$

`objective_weight` is defined in the TimeStruct package (version 0.9.6 in
`Project.toml:31`), not in this repository:

$$w_t = \text{probability}(t)\;\cdot\;\text{discount}(t;\ \text{avg\_year})\;\cdot\;\text{multiple}(t)$$

(`TimeStruct/src/discount.jl:167-204`). [V]

- The discounter is `Discounter(discount_rate(par), 1, periods)` with
  `timeunit_to_year = 1`, so strategic-period durations are read as years
  (`results.jl:961`; the same call is made for the objective). [V]
- `avg_year` discounts to the average of the years inside the strategic period;
  the TimeStruct test shows that the weight equals the discount to the start of
  the strategic period times $\sum_{j=0}^{y-1}(1+r)^{-j}$ over the $y$ years of
  the period, times the multiple, times the probability
  (`test/test_timestruct.jl:21-23`). The sum over years means the period
  **duration is contained in $w_t$**. [V]
- `multiple(t)` scales a modelled representative hour to the number of hours of
  the year it stands for. [V]
- Probability is the scenario probability, equal across scenarios here. [V]

Units: $c$ (EUR/ton) times $x$ (ton/h) times a weight that carries
hours-per-year, years and discounting gives discounted EUR. [I]

Relation to the reserve row: the reserve coefficient
$\text{duration\_strat}\cdot\text{multiple\_strat}$ equals the undiscounted,
unweighted version of $w_t$ without the scenario probability, so cumulative
tonnage is consistent with what is paid for in the objective. [I]

### 1.3 Terminal price and scenario index

The cost lookup uses `context.strategic` and `context.gas`
(`natural_gas.jl:434-435`). With `gasScenarioCount = 1` the scenario index is
always 1. [V]

### 1.4 Reported components

`objective_component_values` returns the value of every component
(`model_definition.jl:934-937`). The runner prints them and stores them in the
run manifest under `objective_components`
(`scripts/run_julia_empire.jl:759-764, 826-852`). The CSV `results_objective.csv`
contains only the total (`results.jl:1785-1789`). [V]

## 2. Cross-sector dependency register

| Dependency | Source | Target | Shared data or variable | Constraint / term | Spatial index | Reference | Type |
|---|---|---|---|---|---|---|---|
| Gas burn for power | Power | Gas | `genOperational` -> `ngForPower`; `genEfficiency` | C1, C9 | (node, generator) | `natural_gas.jl:271-280, 366-403` | direct |
| Pipeline electricity | Gas | Power | `ngTransmission`, $\pi$ | power `flow_balance` | sending node | `natural_gas.jl:165-181`; `model_definition.jl:252-255` | direct |
| Fuel price removal | Gas | Power cost | `genFuelCost` ignored for gas generators | `genMargCost` | generator x period | `utils.jl:268-273` | cost logic |
| Gas-fired emissions | Power | CO2 cap | `genOperational`, emission factor | `node_emission` | node | `model_definition.jl:769-784` | shared data |
| Reformer gas | Hydrogen | Gas | `reformerNaturalGas` | C9; `hydrogen_reformer_natural_gas` | reformer node | `natural_gas.jl:386-392`; `hydrogen.jl:329-337` | direct |
| Transport gas demand gate | Hydrogen flag | Gas | `transport_demand` | C8, C9 | onshore node | `user_interface.jl:292` | build gate |
| Pipeline repurposing | Hydrogen | Gas | `hydrogenRepurposedGasPipelineCapInstalled` | C7; `hydrogen_repurpose_capacity` | gas arc | `natural_gas.jl:342-349`; `hydrogen.jl:612-639` | direct |
| Energy content | Gas parameter | Hydrogen, Industry | $\mu$ | reformer, repurposing, industry factors | n/a | `hydrogen.jl:334, 624`; `industry.jl:391, 398, 687` | shared data |
| Shedding price | Gas parameter | Hydrogen | $\kappa$ | hydrogen shedding terms | onshore node | `hydrogen.jl:890-898` | shared data |
| Industry gas demand | Industry | Gas | `industry_natural_gas_demand` | C9 | node | `natural_gas.jl:402`; `industry.jl:319-344` | direct |
| CCS capture | Power gen | CO2 balance | `genCO2Captured`, `genOperational` | `co2_flow_balance` | onshore node | `hydrogen.jl:521-548` | direct (outside gas) |
| Onshore set | Gas sets | Hydrogen, CO2 | `OnshoreNode` | transport and CO2 balance | onshore node | `hydrogen.jl:153, 521` | shared set |
| Shared objective | Gas | all | components | `@objective` | n/a | `model_definition.jl:202, 920-921` | shared |

Observations: `industry.jl:608` loops over gas nodes while searching for biomass
generators; it concerns biomass, not gas flow, but uses the gas node set as its
node universe. [V]

## 3. Outputs

### 3.1 Implemented reporting

`write_solution_tables` calls `write_natural_gas_csvs` only if gas sets exist
and `ngTerminalImport` is in the model (`results.jl:306-315`). Each table is
produced once and copied to a second Python-style name by
`_write_natural_gas_table` (`results.jl:988-1002`). All tables carry the common
columns `Period, Scenario, WeatherScenario, GasScenario, Season, Hour`, and
values are the raw per-hour solution values (no multiplication by duration or
weight). [V]

| Primary file / Python-style copy | Key columns | Values | Source variables / parameters |
|---|---|---|---|
| `ngTerminalImport.csv` / `results_natural_gas_terminals.csv` | Node, Terminal | `TerminalImport_ton`, `TerminalCapacity_ton_per_h`, `TerminalCost_EUR_per_ton` | $x$, $C^{term}$, $c$ |
| `ngTransmission.csv` / `results_natural_gas_pipeline.csv` | FromNode, ToNode | `PipelineFlow_ton`, `PipelineCapacity_ton_per_h`, `ElectricityUse_MWh` | $f$, $C^{pipe}$, $\pi\cdot f$ |
| `ngForPower.csv` / `results_natural_gas_for_power.csv` | Node, Generator | `NaturalGasForPower_ton`, `Generation_MWh`, `Efficiency` | $q$, $P$, $\eta$ |
| `ngStorage.csv` / `results_natural_gas_storage.csv` | Node | `StorageLevel_ton`, `StorageCharge_ton`, `StorageDischarge_ton`, `StorageCapacity_ton` | $s,u,d,S$ |
| `transportNaturalGas.csv` / `results_transport_naturalGas_operations.csv` | Node (onshore) | `DemandMet_ton`, `DemandShed_ton`, `AnnualDemand_MWh` | $m,h,D$ |
| `naturalGasBalance.csv` / `results_natural_gas_balance.csv` | Node | imports, pipeline in/out, power, storage, transport, `BalanceResidual_ton` | recomputed balance |
| `naturalGasOperationalDuals.csv` | Node | `GasPrice_EUR_per_ton`, `StorageBalanceDual_EUR_per_ton` | duals; written only if `JuMP.has_duals` |

Writers: terminal `results.jl:1046-1085`, pipeline `:1092-1130`, for-power
`:1141-1180`, storage `:1195-1233`, transport `:1243-1285`, balance
`:1302-1386`, duals `:1395-1478`. [V]

Details:
- `ElectricityUse_MWh` is computed in the writer as
  `pipelinePowerDemandPerTon * flow` (`results.jl:1117`). A column named `_ton`
  is the per-hour value. [V]
- The gas price is `dual(natural_gas_flow_balance) / weight`, so it is in
  EUR/ton. The storage dual is multiplied back by `NATURAL_GAS_ROW_SCALE` before
  division by the weight (`results.jl:1439-1456`). [V]
- Hydrogen and Industry results show gas use in their own tables
  (`results_natural_gas_hydrogen.csv`, `results.jl:889-895`; industry tables
  carry `NaturalGas_ton_per_h`, `results.jl:402`). [V]

### 3.2 Available in the JuMP model but not reported

- Duals of C2 (terminal capacity), C3 (reserves, stored in
  `emp[:natural_gas_max_reserves]`), C7 (pipeline capacity) and C6 (storage
  capacity), and reduced costs of all gas variables.
- Cost components other than the totals in the run manifest, for example cost per
  terminal or per node.
- Cumulative reserve use per (node, terminal): it can be recomputed from
  `ngTerminalImport.csv` but is not written. [V by absence]

### 3.3 Potential output issue (unverified)

The transport table writer indexes `transport_met[node, t]` for every onshore
node (`results.jl:1255-1268`). In a gas-only YAML run those sparse variables are
never inserted (`natural_gas.jl:156-161`). I did not run it and did not confirm
how `IndexedVarArray` answers a missing index here. The existing tests use the
lower-level API, where `transport_demand` defaults to true
(`test/test_natural_gas.jl:79, 888`). [U]

## 4. Worked traceability examples

### Example A: gas terminal supply (Germany, DomesticProduction)

1. **Raw data.**
   - `data/full_model_int/Sets/NaturalGasTerminalsOfNode.csv`: pair
     `Germany,DomesticProduction`.
   - `NaturalGas/TerminalCapacity.csv:44`: `Germany,DomesticProduction,1,481.6251318724682`
     (constant across all 7 periods, rows 44-50).
   - `NaturalGas/TerminalCost.csv:44`: `Germany,DomesticProduction,1,1,278.0`.
   - `NaturalGas/Reserves.csv:9`: `Germany,13368462.3634987`.
2. **Readers.** `read_csv.jl:643` (pair set), `:800-829` (capacity), `:831-870`
   (cost), `:787-798` (reserve). Unique-key and non-negativity checks apply.
3. **Parameters.** `terminalCapacity[("Germany","DomesticProduction",1)] =
   481.625`; `terminalCost[("Germany","DomesticProduction",1,1)] = 278.0`;
   `reserves["Germany"] = 13368462.36`.
4. **Validation.** The pair must be in `TerminalsOfNode`; all 7 periods must have
   a capacity and a cost; Germany must have a reserve because the terminal type
   is finite-reserve (`empire_structs.jl:651-660, 678-694`).
5. **Variable.** `ngTerminalImport["Germany","DomesticProduction",t]` $\ge 0$,
   ton/h, inserted at `natural_gas.jl:140-142`.
6. **Constraints.**
   - C2: $x \le 481.625$ for all $t$ (`:281-294`).
   - C3: $\rho\sum \text{dur}\cdot\text{mult}\cdot x \le \rho \cdot 13{,}368{,}462$
     per weather scenario (`:183-242`).
   - C9: $x$ is a supply term in Germany's `natural_gas_flow_balance`
     (`:369-373`).
7. **Objective.** `natural_gas_terminal_import += w_t * 278 * x`
   (`natural_gas.jl:425-436`).
8. **Output.** `ngTerminalImport.csv` row (Germany, DomesticProduction, ...);
   gas price from `naturalGasOperationalDuals.csv`.
9. **Numbers.**
   - Full output: $481.625 \times 8760 = 4{,}219{,}036$ ton/yr.
   - Reserve in years: $13{,}368{,}462 / 4{,}219{,}036 \approx 3.17$ years, so the
     reserve is the binding limit over a multi-decade horizon; the optimiser
     decides when to spend it (using it when alternatives cost more than
     278 EUR/ton). [I from arithmetic; no solve]
   - Price: 278 EUR/ton = 20 EUR/MWh at 13.9 MWh/ton.

### Example B: pipeline electricity use (Germany to Netherlands)

1. **Raw data.**
   - `NaturalGas/PipelineElectricityUse.csv`: `0.02444604019447059` MWh/ton.
   - `NaturalGas/PipelineCapacity.csv:28`: `Germany,Netherlands,2522.332677303271`.
   - The arc is in `Sets/NaturalGasDirectionalLines.csv`.
2. **Readers.** `read_csv.jl:775-785` (scalar, exactly one row) and `:750-773`.
3. **Parameters.** `pipelinePowerDemandPerTon = 0.024446`;
   `pipelineCapacity[("Germany","Netherlands")] = 2522.33`.
4. **Variable.** `ngTransmission["Germany","Netherlands",t]` $\ge 0$ ton/h
   (`natural_gas.jl:143-145`).
5. **Constraint on gas side.** C7: $f \le 2522.33$ (`:336-350`). The gas
   balance (C9) subtracts $f$ at Germany (outflow) and adds it at the
   Netherlands (inflow), with no loss. The electricity rate $\pi$ is **not** used in any
   gas constraint.
6. **Electricity link.** `natural_gas_pipeline_electricity_demand(emp, sets, par,
   "Germany", t)` $= \pi\sum_{j\in\text{out}(\text{Germany})} f_{Gj,t}$
   (`natural_gas.jl:165-181`), subtracted on the left-hand side of Germany's
   power `flow_balance` (`model_definition.jl:252-255`). The sending node pays.
7. **Objective.** No gas cost term. The electricity is paid indirectly: more
   load at Germany raises the power-side generation cost and may force
   shedding (`load_shedding` component).
8. **Output.** `ngTransmission.csv`: `PipelineFlow_ton`, `PipelineCapacity`,
   `ElectricityUse_MWh = π · flow` (`results.jl:1117`).
9. **Numbers.** At full flow: $2522.33 \times 0.02444604 = 61.66$ MW (about
   35.1 GWh/h of gas energy is moved, so about 0.18 % of the energy carried).
   [V arithmetic]
10. **Cross-sector.** Direct Gas to Power coupling; it exists only because both
    sectors share the node name `Germany`.

### Example C: transport gas demand (Germany, period 1)

1. **Raw data.**
   - `Transport/NaturalGasDemand.csv:8`: `Germany,1,506030485.88991034`
     (MWh/yr). Period 2 is 392,156,563.4.
   - `Transport/CurtailCost.csv`: `100000`.
2. **Readers.** `read_csv.jl:872-900` and `:902-912`.
3. **Parameters.** `transportDemand[("Germany",1)] = 506,030,485.89`;
   $\kappa = 100000$; $\mu = 13.9$ (code constant).
4. **Transformation.** $D/(8760\mu) = 506{,}030{,}485.89 / 121{,}764 = 4155.83$
   ton/h (arithmetic checked), applied in every operational period of strategic
   period 1.
5. **Variables.** `transportNaturalGasDemandMet["Germany",t]` and
   `...Shed["Germany",t]`, ton/h, **only if `transport_demand` is true**, i.e.
   `hydrogen: true` in the YAML path (`user_interface.jl:292`).
6. **Constraints.**
   - C8: $m + h \ge 4155.83$ (`natural_gas.jl:351-365`).
   - C9: $m$ is a consumption term in Germany's gas balance (`:398-402`).
7. **Objective.** `natural_gas_transport_shedding += w_t * 100000 * h`
   (`natural_gas.jl:437-445`). Hydrogen shedding terms use the same constant
   (`hydrogen.jl:890-898`).
8. **Output.** `transportNaturalGas.csv`: `DemandMet_ton`, `DemandShed_ton`,
   `AnnualDemand_MWh` (`results.jl:1243-1285`).
9. **Scale check.** The shedding price 100,000 per ton is about 7,194 per MWh
   if read per energy ($100000/13.9$), well above the 278 EUR/ton gas price, so
   shedding is only a last resort. [I from arithmetic]
10. **Remarks.**
    - In gas-only YAML runs this demand is read and validated but never used.
    - Demand decreases from 5.06e8 MWh/yr (period 1) to zero in periods 6 and 7
      for Germany, as for most nodes. [V]
    - The CSV label is EUR/MWh; the model applies it per ton. [U]

## 5. NUTS2 observations (no design)

- The objective is spatially blind: gas cost has no node weighting, so the
  spatial structure enters only through which nodes have terminals, capacities
  and demand. [V]
- A transport demand total per onshore node would have to be allocated by some
  rule if nodes were refined; the model contains no such rule. [I]
- Pipeline electricity is charged to the power node of the same name; a regional
  split of the electricity side would change where that load is placed. [I]
- Hard-coded constants shared with other sectors: 13.9 and the shedding price. [V]

## 6. Open questions

1. Curtailment unit (EUR/MWh label versus per-ton use), and its shared use by
   Hydrogen. [U]
2. Italy's reserve applied per terminal. [U]
3. Whether `results.jl:1255-1268` fails in gas-only YAML runs (section 3.3). [U]
4. Whether Hydrogen-dependent transport demand matches InternalEMPIRE's
   scenario handling for G > 1. Blocked by validation (`empire_structs.jl:611`). [U]
5. Where `leap_years_investment` becomes `duration_strat`
   (`model_definition.jl:70`). [I]
