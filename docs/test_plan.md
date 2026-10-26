# Test plan: Bizzdesign data flow scripts

This plan tests all scripts in a **separate, empty model**, so the real model is not touched.
`setup_test_project.script` builds the test fixture: one view per scenario, with its own elements, relations and access types.

Expected results come from a simulation of the shared path engine on the same fixture.
A result that differs from this plan is either a script bug or a Bizzdesign difference; please report it with the output table.

---

## 1. Setup

### 1.1 Create the test model
1. In Enterprise Studio, create a **new empty ArchiMate model** (e.g. `Data flow scripts - TEST`).
2. Create one ArchiMate view named **`Seed`** and leave it empty.

### 1.2 Add the scripts to the test model
Scripts saved as viewpoints are stored in the model, so they must also be created in the test model.
Copy the text of each file into a new script/viewpoint:

| File | Viewpoint name | Output presentation type | Context-menu enabler |
|---|---|---|---|
| `data_flow_path_engine.script` | **exactly** `data_flow_path_engine` | Table | – |
| `setup_test_project.script` | any | Table | – |
| `associate_data_n_flow.script` | any | Table | – |
| `data_flow_consistency_check.script` | any | Table | – |
| `data_flow_consistency_fix.script` | any | Table | – |
| `data_lineage.script` | any (MODE = "table") | Table | – |
| `data_lineage.script` copy with `MODE = "highlight";` | any | View: Highlight view | – |
| `data_impact_analysis.script` | any (MODE = "table") | Table | – |
| `data_impact_analysis.script` copy with `MODE = "highlight";` | any | View: Highlight view | – |
| `routing_functions.script` | any | Table | – |
| `manage_data_roles.script` | any | Table | `manage_data_roles_contextmenu.script` |
| `create_access_relations.script` | any | Table | `create_access_relations_contextmenu.script` |
| `interface_report.script` | any | Table | `interface_report_contextmenu.script` |
| `flow_query_table.script` | any | Table | `flow_query_table_contextmenu.script` |
| `flow_data_tooltip.script` | any | View: Tooltip view | – |
| `flow_data_report.script` | any | Table | – |
| `probe_view_items.script` (optional) | any | Table | – |

### 1.3 Build the fixture
1. Select the `Seed` view in the model browser.
2. Run `setup_test_project.script`.
3. If it reports *"Cannot find where to create elements of type …"*:
   - Place one Application Component, one Application Function and one Data Object (any names) on the `Seed` view.
   - Run the script again. The seed elements are not used by any test. Delete them from the model afterwards; otherwise the seed Data Object adds 2 issues (No producer, No consumer) to every check.
4. If it reports *"The test fixture already exists"*, the model already has the fixture. Start from a new empty model.

| ID | Check | Expected |
|---|---|---|
| S-01 | Last output row | `Done` – 125 elements, 153 relations, 23 views, `0 problem(s)` (151 relations / 2 problems if the E13 lines are refused, see note below) |
| S-02 | Views created next to `Seed` | 23 views, `E01 …` to `A01 …` (list in section 2) |
| S-03 | Open `E04 Processor required` | Four functions, one data object, five flows, four access relations, laid out automatically |
| S-04 | Access type of `E04 Enrich A` → `E04 Data` (properties) | `read/write` |
| S-05 | `K01 Consistency check and fix` view | Two association lines from flows to `K01 Data` |

> **Acceptable setup problems:** `E13` deliberately contains two unusual relations: a Component → Data Object access and a Function → Component flow.
> If the metamodel refuses them, the setup lists those two lines as `NOT CREATED`. E13 is still a valid test: the engine must ignore non-function participants either way.

**Do not run `data_flow_consistency_fix.script` or `associate_data_n_flow.script` with nothing selected** until section 5.
Running them on everything would change the fixtures of K01, K02 and R01 before their own tests.

---

## 2. Fixture overview

Each scenario has its own elements, all prefixed with the scenario code.
Arrows are Flow relations. Roles come from the access type: **W** = write (producer), **RW** = read/write (processor), **R** = read (consumer), **N** = none (excluded), **–** = no access (router).

| Code | View | Model | Purpose |
|---|---|---|---|
| E01 | Simple chain with router | Produce(W) → Route(–) → Consume(R); each function in its own component | Basic path, router, component chain |
| E02 | Shortest path without processors | Produce(W) → Consume(R); Produce → Detour(–) → Consume | Only the shortest path is kept |
| E03 | Equal-length alternatives | Produce(W) → Route A → Consume(R); Produce → Route B → Consume | Ties are all kept. Two *different* routers, so it respects the one-flow-per-direction convention (redundant route, low priority) |
| E04 | Processor required | Produce(W) → Consume(R); Produce → Enrich A(RW) → Consume; Produce → Enrich B(RW) → Consume | Direct path rejected because another path to the same consumer passes a processor; alternative processors are both kept |
| E05 | Processor not on any path | Produce(W) → Consume(R) → Validate(RW) | The processor is only after the consumer: the direct path is kept and the check reports `Processor off path` |
| E06 | Excluded function blocks a path | Produce(W) → Archive(N) → Consume(R); Produce → Route 1 → Route 2 → Consume | Excluded function blocks; the longer path is used. *Optional* – access type none is rare |
| E07 | Consumer stops the path | Produce(W) → Consume 1(R) → Consume 2(R) | Path ends at the first consumer |
| E08 | Several consumers no processor | Produce(W) → Consume Near(R); Produce → Route → Consume Far(R) | Shortest path to **each** consumer is kept |
| E09 | Several consumers with a processor | Produce(W) → Transform(RW) → Consume Near(R); Transform → Route → Consume Far(R) | With a processor both consumers are reached |
| E10 | Two producers (redundancy) | Produce 1(W) → Consume(R); Produce 2(W) → Route → Consume | Allowed; check shows `Note: several producers`; each producer is walked |
| E11 | Producer in the middle (rule violation) | Produce 1(W) → Produce 2(W) → Consume(R) | `Producer downstream` issue (Produce 2 should be read/write); a path never passes another producer |
| E12 | Cycle | Produce(W) → Loop A → Loop B → Loop A …; Loop B → Consume(R) | No endless loop |
| E13 | Non-function participants ignored | Produce(W) → Consume(R); Produce → *Data Owner System* (component, has W access) | Only functions get roles or are walked |
| E14 | Missing producer or consumer | Two data objects: one with only a consumer, one with only a producer | `No producer` / `No consumer` |
| E15 | Flow carrying two data objects | Produce → Consume, both W/R on `E15 Data 1` and `E15 Data 2` | Tooltip and flow table with 2 data objects |
| E16 | Processor bypass | Produce → Process → Route → Old Consume → New Consume, plus the shortcut Produce → Route. Process and Old Consume are read/write, New Consume is read | A path that skips a processor is dropped when another path passes it as well (case reported from the real model) |
| E17 | Processor required per consumer | Produce(W) → Consume Direct(R); Produce → Process(RW) → Consume Processed(R) | The processor rule applies per consumer: the direct path to Consume Direct is kept |
| E18 | Parallel flows and repeated access | Two flows Produce(W) → Consume; Consume has two access relations (R and RW) | `Parallel flows` and `Several access relations` issues |
| K01 | Consistency check and fix | Produce(W) → Route → Consume(R); Produce → Side. Pre-associated: Produce→Route (correct), Produce→Side (extra) | Missing + extra association, both association directions |
| K02 | Role change workflow | Produce(W) → Old Consume(R) → New Consume(–); Produce→Old Consume associated | Changing roles, then fix |
| I01 | Impact analysis and routing | Produce(W) → Transform(RW) → Route Hub → Consume(R); Transform → Route Alt → Consume. Hub System and Alt System are both inside Platform | Lost vs Degraded, component scope, nesting |
| R01 | Interface report | Platform contains Sub A (Function A, with sub-function A.1) and Sub B (Function B). External System (External Function), Unowned Function, Isolated System | Boundary crossing, sub-components, sub-functions, component-level flow |
| A01 | Create access relations | Function 1, 2, 3; `A01 Existing Data` (Function 1 has read access) | Creating and updating access relations |

---

## 3. Path engine and consistency check (baseline)

### T-ENG-00 Engine run directly
| Step | Expected |
|---|---|
| Run `data_flow_path_engine` on its own, with any selection | Error: `data_flow_path_engine: variable dataObject must be an Application Data Object.` (proves the engine refuses to run without the variable) |

### T-CHK-01 Baseline check (nothing associated yet)
Run `data_flow_consistency_check.script` with **nothing selected**. Row order may differ.

| Data object | Expected findings |
|---|---|
| E01 Data | Missing association ×2: Produce→Route, Route→Consume |
| E02 Data | Missing association ×1: Produce→Consume (**not** Produce→Detour, Detour→Consume); `Note: ignored route` … via E02 Detour |
| E03 Data | Missing association ×4: Produce→Route A, Route A→Consume, Produce→Route B, Route B→Consume |
| E04 Data | Missing association ×4: Produce→Enrich A, Enrich A→Consume, Produce→Enrich B, Enrich B→Consume (**not** Produce→Consume) |
| E05 Data | Processor off path: E05 Validate; Missing association ×1: Produce→Consume |
| E06 Data | Missing association ×3: Produce→Route 1, Route 1→Route 2, Route 2→Consume; `Note: excluded function` E06 Archive |
| E07 Data | Unreached consumer: E07 Consume 2, detail "Reachable only through consumer E07 Consume 1; should E07 Consume 1 be read/write (processor)?"; Missing association: Produce→Consume 1 |
| E08 Data | Missing association ×3: Produce→Consume Near, Produce→Route, Route→Consume Far |
| E09 Data | Missing association ×4: Produce→Transform, Transform→Consume Near, Transform→Route, Route→Consume Far |
| E10 Data | `Note: several producers`; Missing association ×3: Produce 1→Consume, Produce 2→Route, Route→Consume |
| E11 Data | `Note: several producers`; Producer downstream: E11 Produce 2 (from E11 Produce 1); Missing association ×1: Produce 2→Consume (**not** Produce 1→Produce 2) |
| E12 Data | Missing association ×3: Produce→Loop A, Loop A→Loop B, Loop B→Consume (**not** Loop B→Loop A) |
| E13 Data | Missing association ×1: Produce→Consume (the component is not a producer; the flow to the component is ignored) |
| E14 Data No Producer | No producer |
| E14 Data No Consumer | No consumer |
| E15 Data 1 / E15 Data 2 | Missing association: Produce→Consume (each) |
| E16 Data | Missing association ×4: Produce→Process, Process→Route, Route→Old Consume, Old Consume→New Consume (**not** Produce→Route) |
| E17 Data | Missing association ×3: Produce→Consume Direct, Produce→Process, Process→Consume Processed |
| E18 Data | Several access relations: E18 Consume; Parallel flows (1 row); Missing association ×2 (one per parallel flow) |
| K01 Data | Missing association: Route→Consume; Extra association: Produce→Side |
| K02 Data | OK – 1 flow(s) consistent |
| I01 Data | Missing association ×5: Produce→Transform, Transform→Route Hub, Transform→Route Alt, Route Hub→Consume, Route Alt→Consume |
| R01 Data | No producer; No consumer; Extra association: External Function→Function A (expected: R01 is only for the interface report) |
| A01 Existing Data | No producer |
| (last row) | `Total`, 60 issue(s), 4 note(s) (notes: E02, E06, E10, E11) |

### T-CHK-02 Check with a selection
| Step | Expected |
|---|---|
| Select `E02 Data` on its view and run the check | Only the E02 row(s), Total = 1 issue, 1 note |
| Select `E01 Data` and `E04 Data` (Ctrl+click) | E01 and E04 rows only, Total = 6 |

### T-CHK-02b Longer detour through the same downstream processor
Use a separate manual scenario (not included in the setup totals):
- Roles: P = write, X = read/write, C = read; U and R have no access.
- Flows: P→R, P→U, U→R, R→X, X→C.
- Associate only P→R and R→X initially.
- Check: only X→C is `Missing association`; P→U and U→R are not missing.
  The longer route via U, R, X appears as `Note: ignored route`.
- Associate/fix, then check: `OK`, 3 flows consistent, plus the ignored-route note.
- Add a second equal-length route P→V→X (V has no access): both shortest
  routes through X are kept, including their shared X→C flow.
- Also rerun E04 and E16: alternative processors remain eligible, and the
  processor-bypassing shortcut remains excluded.

---

## 4. Associate data and flow (`associate_data_n_flow.script`)

For each data object below: select it on its view, run the script, and compare the table.
The flows in the second column must get the status `association created`; the flows in the third column must **not** appear.

| ID | Data object | Associations created (exact set) | Must NOT be associated |
|---|---|---|---|
| T-ASC-01 | E01 Data | Produce→Route, Route→Consume | – |
| T-ASC-02 | E02 Data | Produce→Consume | Produce→Detour, Detour→Consume |
| T-ASC-03 | E03 Data | Produce→Route A, Route A→Consume, Produce→Route B, Route B→Consume | – |
| T-ASC-04 | E04 Data | Produce→Enrich A, Enrich A→Consume, Produce→Enrich B, Enrich B→Consume | Produce→Consume |
| T-ASC-05 | E05 Data | Produce→Consume | Consume→Validate |
| T-ASC-06 | E06 Data | Produce→Route 1, Route 1→Route 2, Route 2→Consume | Produce→Archive, Archive→Consume |
| T-ASC-07 | E07 Data | Produce→Consume 1 | Consume 1→Consume 2 |
| T-ASC-08 | E08 Data | Produce→Consume Near, Produce→Route, Route→Consume Far | – |
| T-ASC-09 | E09 Data | Produce→Transform, Transform→Consume Near, Transform→Route, Route→Consume Far | – |
| T-ASC-10 | E10 Data | Produce 1→Consume, Produce 2→Route, Route→Consume | – |
| T-ASC-11 | E11 Data | Produce 2→Consume | Produce 1→Produce 2 |
| T-ASC-12 | E12 Data | Produce→Loop A, Loop A→Loop B, Loop B→Consume | Loop B→Loop A |
| T-ASC-13 | E13 Data | Produce→Consume | Produce→Data Owner System |
| T-ASC-14 | E14 Data No Producer, then E14 Data No Consumer | none – `No producer-to-consumer path found` (both) | Produce Only→Consume Only |
| T-ASC-15 | E15 Data 1, then E15 Data 2 | Produce→Consume (each) | – |
| T-ASC-15b | E16 Data | Produce→Process, Process→Route, Route→Old Consume, Old Consume→New Consume | Produce→Route |
| T-ASC-15c | E17 Data | Produce→Consume Direct, Produce→Process, Process→Consume Processed | – |
| T-ASC-15d | E18 Data | both parallel Produce→Consume flows | – |
| T-ASC-15e | E02 Data (in T-ASC-02) | The table also shows `Note: longer route from E02 Produce to E02 Consume via E02 Detour` | – |
| T-ASC-15f | E07 Data (in T-ASC-07) | The table also shows `Warning: consumer E07 Consume 2 is reachable only through consumer E07 Consume 1` | – |
| T-ASC-15g | E11 Data (in T-ASC-11) | The table also shows `Warning: producer E11 Produce 2 is reached from producer E11 Produce 1` | – |
| T-ASC-16 | I01 Data | the 5 flows of the two paths | – |
| T-ASC-17 | E01 Data again | both rows `already associated`, Total `Created 0` | – |
| T-ASC-18 | Select a function (not a data object) | Error `Select a Data Object.` | – |
| T-ASC-19 | Nothing selected | Error `Select one Data Object in the view and run the script again.` | – |

### T-CHK-03 Check after association
Run the check with nothing selected. Expected:
- **OK** for E01, E02, E03, E04, E06, E08, E09, E10, E12, E13, E15 Data 1/2, E16, E17, K02, I01 (E02, E06 and E10 still show their note rows).
- **Remaining findings**, all expected:
  - E05: processor off path.
  - E07: Consume 2 unreached.
  - E11: producer downstream.
  - E18: several access relations, parallel flows.
  - E14: no producer / no consumer.
  - K01: missing and extra association.
  - R01: no producer, no consumer, extra association.
  - A01: no producer.
- Total = **13** issue(s), 4 note(s).

---

## 5. Consistency fix (`data_flow_consistency_fix.script`), scenario K01

Select **`K01 Data`** for every test.

| ID | Steps | Expected |
|---|---|---|
| T-FIX-01 | Run, answer **Cancel** at the first prompt | Prompt "Add 1 missing flow association(s)?". After Cancel nothing changes; the check still shows the same 2 K01 findings |
| T-FIX-02 | Run, answer **No**, then **No** | Second prompt "Delete 1 extra flow association(s)…". Table: `K01 Route -> K01 Consume` skipped (not confirmed); `K01 Produce -> K01 Side` kept (not confirmed). Model unchanged |
| T-FIX-03 | Run, answer **Yes**, then **Yes** | Table: Route→Consume `added`; Produce→Side `deleted`. The association line to Side disappears from the K01 view |
| T-FIX-04 | Run the check on K01 Data | `OK – 2 flow(s) consistent` |
| T-FIX-05 | Run the fix again | Message "All flow associations are consistent. Nothing to change." |

*T-FIX-03 also covers both association directions. The setup tries to create the Produce→Side association from the data object to the flow, and the fix must delete it whichever direction it ended up with.*

---

## 6. Manage data roles (`manage_data_roles.script`), scenario K02

If a current consumer is explicitly selected, only the selected functions are reviewed, without a neighbour question. Otherwise, current consumers, explicitly selected functions and optionally the consumers' immediate flow neighbours are reviewed. Neighbours are one hop away in either direction; discovery does not expand through them. Other access relations stay unchanged. Each function has a Yes/No/Cancel question followed by the native access-type dropdown when Yes is chosen.

### Main workflow: a new function becomes the consumer
| ID | Steps | Expected |
|---|---|---|
| T-ROL-01 | Select `K02 Data` and `K02 New Consume`, run | Optional neighbour question offers only `K02 Produce` (an immediate neighbour of Old Consume). Answer **No** |
| T-ROL-02 | Dialogs 1/2 – 2/2 (each: Yes/No question, then the access-type dropdown) | Only `K02 Old Consume` (read → read/write) and selected `K02 New Consume` (no access → read) are reviewed. Produce is not prompted |
| T-ROL-03 | Confirmation | "Apply 2 role change(s)?" (information icon, no warning) → OK |
| T-ROL-04 | Result table | Old Consume read → read/write `access type changed`; New Consume no access → read `access relation created`; Produce is absent from the table and retains write access. No missing-producer warning; last row `Next: run data_flow_consistency_fix.script…` |
| T-ROL-05 | Run the check on K02 Data | Missing association: Old Consume→New Consume (1 issue) |
| T-ROL-06 | Run the fix on K02 Data, **Yes** | Old Consume→New Consume `added`; the check then shows OK – 2 flow(s) |

### Follow-up: the old consumer becomes a pure router
| ID | Steps | Expected |
|---|---|---|
| T-ROL-07 | Select K02 Data, run; answer **Yes** to review immediate neighbour Old Consume. Keep New Consume as read; Old Consume → answer **No** to the access question | Old Consume access relation `deleted`; Produce is not reviewed and remains write |
| T-ROL-08 | Run the check on K02 Data | OK – 2 flow(s) (the path is unchanged; Old Consume is now a router) |
| T-ROL-09 | `routing_functions.script` with K02 Data selected | One row: router `K02 Old Consume`, component `K02 Legacy System`, receives from `K02 Produce`, forwards to `K02 New Consume` |

### Follow-up: the old function is excluded (optional – access type none is rare)
| ID | Steps | Expected |
|---|---|---|
| T-ROL-10 | Run on K02 Data. Old Consume is not a role holder any more, so answer **Yes** to the neighbour question; Old Consume → **Yes**, choose **none** | Access relation `created` with type none |
| T-ROL-11 | Check on K02 Data | Unreached consumer: New Consume; Extra association ×2: Produce→Old Consume, Old Consume→New Consume |
| T-ROL-12 | Fix, **Yes** to delete | Both associations deleted; the check then shows only `Unreached consumer` |

### Negative and edge cases
| ID | Steps | Expected |
|---|---|---|
| T-ROL-20 | Select `E01 Data` and `E02 Data`, run | Error `Select only one Data Object.` |
| T-ROL-21 | Nothing selected, run | Data Object picker; Cancel → stops, nothing changed |
| T-ROL-22 | Select E01 Data; at a question answer **Yes**, then Cancel the dropdown | Stops; nothing changed |
| T-ROL-23 | Cancel at the Yes/No question 2 | Stops; nothing changed (check that the access types are unchanged) |
| T-ROL-24 | Answer Yes and keep all pre-filled values | Message `No role changes.` |
| T-ROL-25 | `E14 Data No Producer`, set E14 Consume Only to read (unchanged) and E14 Produce Only (neighbour) to read | Confirmation shows the **warning** icon: "… will have 0 producer(s) and 2 consumer(s) (rule: at least one of each)." Cancel → nothing changed |
| T-ROL-26b | `E01 Data`, set E01 Consume to **write** (no consumer left) | Warning: "… will have 2 producer(s) and 0 consumer(s) (rule: at least one of each)." Cancel → nothing changed. (Two producers alone no longer warn.) |
| T-ROL-26 | Select `E01 Data` + `A01 Function 1`, run | `A01 Function 1` is offered as an extra participant (no access) |
| T-ROL-27 | Context menu | Enabled for exactly 1 Data Object (+ optional functions); disabled for 2 Data Objects or when a component is selected |
| T-ROL-28 | Select E16 Data and E16 New Consume; accept reviewing neighbours, then keep every role | Only New Consume and its immediate neighbour Old Consume are prompted. Produce, Process and Route are not prompted. Message `No role changes.` |
| T-ROL-29 | Select E14 Data No Consumer alone, run | Error says there is no current consumer or selected Application Function; no changes. Select its producer together with the Data Object and rerun to edit it explicitly |
| T-ROL-30 | Select E01 Data alone; decline neighbours and keep Consume as read | Only Consume is prompted; Produce and Route retain their existing roles. Message `No role changes.` |
| T-ROL-31 | Before the K02 role change, select K02 Data, K02 Old Consume and K02 New Consume; run | No neighbour question. Only the two selected functions are prompted. Set Old Consume to read/write and New Consume to read; producer remains write and confirmation has no missing-producer warning |
| T-ROL-32 | Select E08 Data and E08 Consume Near; run and keep read | Only Consume Near is prompted, with no neighbour question. Consume Far is not prompted and retains read; message `No role changes.` |

---

## 7. Analysis reports

### Data lineage (`data_lineage.script`, Table)
| ID | Selection | Expected |
|---|---|---|
| T-LIN-01 | E01 Data | Function rows: E01 Produce (E01 Source System) producer; E01 Consume (E01 Target System) consumer; E01 Route (E01 Middleware) router. Path 1: `E01 Produce -> E01 Route -> E01 Consume`, components `E01 Source System -> E01 Middleware -> E01 Target System`, `2 flow(s)` |
| T-LIN-02 | E06 Data | E06 Archive listed with role `excluded`; Route 1 and Route 2 as routers; 1 path with 3 flows |
| T-LIN-03 | E05 Data | Roles (Produce, Consume, Validate) and the row `No producer-to-consumer path found` |
| T-LIN-04 | E03 Data + E04 Data | 2 paths each; E04 paths go through Enrich A and Enrich B |
| T-LIN-05 | Highlight copy, I01 Data on the I01 view | Highlighted: I01 Data, all 5 functions, the 5 flows, the 3 access relations and the association lines |
| T-LIN-06 | Nothing selected | Error `Select one or more Data Objects …` |

### Routing functions (`routing_functions.script`)
T-RTE-01: run with nothing selected (after section 4; before T-ROL-07). Expected rows, in any order:

| Router | Component | Data object | Receives from | Forwards to |
|---|---|---|---|---|
| E01 Route | E01 Middleware | E01 Data | E01 Produce | E01 Consume |
| E03 Route A | (none) | E03 Data | E03 Produce | E03 Consume |
| E03 Route B | (none) | E03 Data | E03 Produce | E03 Consume |
| E06 Route 1 | (none) | E06 Data | E06 Produce | E06 Route 2 |
| E06 Route 2 | (none) | E06 Data | E06 Route 1 | E06 Consume |
| E09 Route | (none) | E09 Data | E09 Transform | E09 Consume Far |
| E08 Route | (none) | E08 Data | E08 Produce | E08 Consume Far |
| E10 Route | (none) | E10 Data | E10 Produce 2 | E10 Consume |
| E12 Loop A | (none) | E12 Data | E12 Produce | E12 Loop B |
| E12 Loop B | (none) | E12 Data | E12 Loop A | E12 Consume |
| K01 Route | (none) | K01 Data | K01 Produce | K01 Consume |
| I01 Route Hub | I01 Hub System | I01 Data | I01 Transform | I01 Consume |
| I01 Route Alt | I01 Alt System | I01 Data | I01 Transform | I01 Consume |

Not listed: E02 Detour, E06 Archive (none ≠ router), E12 Loop B → Loop A as a "forwards to".
T-RTE-02: select only `E02 Data` → `No routing-only functions found`.

### Impact analysis (`data_impact_analysis.script`, Table)
Select the element(s) on the view and run.

| ID | Selection | Expected rows (Data object / Consumer / Impact / Caused by) |
|---|---|---|
| T-IMP-01 | I01 Route Hub | I01 Data / I01 Consume / **Degraded (alternative path exists)** / I01 Route Hub (router) |
| T-IMP-02 | I01 Hub System (component) | same as T-IMP-01, through component → assigned function |
| T-IMP-03 | I01 Hub System + I01 Alt System | I01 Data / I01 Consume / **Lost** / I01 Route Hub (router), I01 Route Alt (router) |
| T-IMP-04 | I01 Platform | same as T-IMP-03, through composition → sub-components → functions |
| T-IMP-05 | I01 Transform | **Lost** / I01 Transform (processor) |
| T-IMP-06 | I01 Produce | **Lost** / I01 Produce (producer) |
| T-IMP-07 | I01 Consume | `No downstream impact found` (a failing consumer is not reported as impacted) |
| T-IMP-08 | E03 Route A | E03 Consume **Degraded** |
| T-IMP-09 | E03 Route A + E03 Route B | E03 Consume **Lost** |
| T-IMP-10 | E04 Enrich A | E04 Consume **Degraded** / E04 Enrich A (processor) |
| T-IMP-11 | E10 Produce 2 (rule-violating model, low priority) | E10 Consume **Degraded** / E10 Produce 2 (producer) |
| T-IMP-11b | E08 Route | E08 Consume Far **Lost** / E08 Route (router); E08 Consume Near **not** listed |
| T-IMP-12 | E02 Detour | `No downstream impact found` (Detour is not on a path) |
| T-IMP-13 | Highlight copy, I01 Hub System on the I01 view | Highlighted: Hub System, Route Hub, the flows of the hit path (Produce→Transform, Transform→Route Hub, Route Hub→Consume), I01 Consume and I01 Data |
| T-IMP-14 | Nothing selected | Error `Select one or more Application Functions or Application Components …` |

### Interface report (`interface_report.script`), scenario R01
Columns: Direction / Internal element / Flow / External element / External component / Data objects. The flow name is `(unnamed)`.

| ID | Selection | Expected rows |
|---|---|---|
| T-IFR-01 | R01 Platform | Outgoing: Function B → External Function (R01 External System). Outgoing: Function B → Unowned Function (`(none)`). Incoming: Function A ← External Function (R01 External System), data `R01 Data`. Incoming: R01 Platform ← R01 External System (R01 External System). **Not listed:** Function A.1 → Function B (internal to the Platform) |
| T-IFR-02 | R01 Sub A | Outgoing: Function A.1 → Function B (R01 Sub B). Incoming: Function A ← External Function (R01 External System), data `R01 Data` |
| T-IFR-03 | R01 Sub B | Outgoing: Function B → External Function; Function B → Unowned Function (`(none)`). Incoming: Function B ← Function A.1, external component **R01 Sub A** (found through the sub-function composition) |
| T-IFR-04 | R01 External System | Outgoing: External Function → Function A (R01 Sub A), data R01 Data. Outgoing: R01 External System → R01 Platform (R01 Platform). Incoming: External Function ← Function B (R01 Sub B) |
| T-IFR-05 | R01 Isolated System | `-` / `No interfaces found` |
| T-IFR-06 | R01 Platform + R01 Isolated System | Both blocks |
| T-IFR-07 | A function only | Error `Select one or more Application Components …` |
| T-IFR-08 | Context menu | Enabled only when the selection contains only components |

---

## 8. Other scripts

### Create access relations (`create_access_relations.script`), scenario A01
Run each test on the A01 view.

| ID | Steps | Expected |
|---|---|---|
| T-CAR-01 | Select Function 2 + Function 3 → **Yes** (new) → name `A01 New Data` → Function 2 **write**, Function 3 **read** | Table: both `created`, On view `shown`; `A01 New Data` is placed to the right of the functions with two access lines; the access type is shown correctly |
| T-CAR-02 | Select Function 1 + Function 3 + `A01 Existing Data` | No create/select question. The Function 1 prompt is pre-filled with **read** → choose read/write; Function 3 → read. Table: Function 1 `updated`, Function 3 `created` |
| T-CAR-03 | Select Function 2 → **No** → pick `A01 Existing Data` → write | Function 2 `created` |
| T-CAR-04 | Select Function 1 → Yes → name `A01 Cancel Test` → **Cancel** at the type prompt | `Cancelled, nothing changed.` and **no** `A01 Cancel Test` in the model |
| T-CAR-05 | Yes → empty name | Stops, nothing created |
| T-CAR-06 | Select a component only | Error `Select one or more Application Functions …` |
| T-CAR-07 | Context menu | Enabled for functions and data objects only; disabled when a component or flow is in the selection |

### Flow data tooltip / table / report (after section 4)
| ID | Steps | Expected |
|---|---|---|
| T-TIP-01 | Activate the tooltip viewpoint on the E15 view, hover the flow | `E15 Data 1, E15 Data 2` |
| T-TIP-02 | E02 view, hover Produce→Detour | `No associated data object` |
| T-TIP-03 | E04 view, hover Produce→Enrich A | `E04 Data` |
| T-FQT-01 | Select the E15 flow, run `flow_query_table.script` | 2 rows: `E15 Produce -> E15 Consume` with E15 Data 1 and E15 Data 2 |
| T-FQT-02 | Select a function | Error `Select one Flow relation.`; the context menu is disabled for non-flows |
| T-FDR-01 | Run `flow_data_report.script` | One row per flow and data object: the E15 flow appears twice (`Associated`); E02 Produce→Detour appears once with `No associated data object` |
| T-FDR-02 | Open a second model in the same package and run the report from the test model | Only flows of the test model are listed (all source/target names start with a scenario code) |

### Probe (optional)
| ID | Steps | Expected |
|---|---|---|
| T-PRB-01 | On any view, select an element, a relation line and (if available) a drawn graphic line → run `probe_view_items.script` | Type names of each item; send the output (needed for the synthetic component view) |

---

## 9. Modelling conventions and remaining points

Agreed rules (see also `flow_path_ambiguity_workbook.html`):

| # | Rule | How the scripts apply it | Test |
|---|---|---|---|
| 1 | At most one flow per direction between two functions | Check: `Parallel flows` issue | E18 |
| 2–4 | Alternative routes without a processor: the shortest route(s) per producer–consumer pair are associated; ties are all kept | Engine; the check and the association script show `Note: ignored route` for longer routes (was a processor forgotten?) | E02, E03, E08 |
| 5, 8 | Several producers are allowed (e.g. redundancy). Per producer–consumer pair: a route through a processor is preferred, otherwise the shortest | Engine; check shows `Note: several producers` | E10, E17 |
| 6 | A producer must not be downstream of another producer; it should be read/write | Check: `Producer downstream` issue | E11 |
| 7 | A consumer that forwards the data should be read/write | Check: the `Unreached consumer` detail names the consumer it is behind | E07 |
| 9 | A route whose processor set is a strict subset of another route's is dropped; alternative processor sets are kept. For an identical processor set, only the shortest routes are kept, including ties | Engine | E04, E16, T-CHK-02b |
| 10 | At least one producer and one consumer | Check: `No producer` / `No consumer`; `manage_data_roles.script` warns | E14, T-ROL-25 |
| 12 | Access type none = the function does not carry the Data Object (cuts a route) | Engine; check shows `Note: excluded function` | E06 |
| 13 | One access relation per function and Data Object | Check: `Several access relations` issue | E18 |
| 14 | Cycles: only the producer–consumer route is associated | Engine never visits a function twice | E12 |
| 19 | The analysis is model-wide, not limited to a view. Only Application Functions are walked (no junctions, processes, events) | Engine | E13 |

Not implemented (possible later): a per-Data-Object route policy (`shortest` / `all`) instead of always keeping the shortest route.

Remaining behaviour to be aware of:
1. **E07 – a consumer stops the path.** A function with read access is an end point; if it forwards the data, make it read/write.
2. **E05 – a processor that is not on any route** is reported as `Processor off path`; the other routes are still associated.

---
## 10. Results log

| Test ID | Pass / Fail | Actual result / error message | Date |
|---|---|---|---|
| S-01 | | | |
| T-ENG-00 | | | |
| T-CHK-01 | | | |
| … | | | |
