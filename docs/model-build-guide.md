# Plant Simulation 2404 — Click-by-Click Guide
## Rework Paint Shop (Exam Task, Group 5 parameters)

This guide assumes you have **never opened Plant Simulation before**. Follow it top to bottom. Every value below comes from your file `20260511_Parameter_ExamTask_G5.xlsx`.

---

## 0. Your Group 5 numbers (so you can double-check against Excel)

**Target:** 53 units per hour (uph) at the Drain ("Ship").

**Color mix:** red 15 %, white 60 %, black 25 %.

**Stations**

| Station  | Capacity | Takt time | Availability | MTTR |
|----------|----------|-----------|--------------|------|
| TC_Entry | 1        | 62 s      | 98.5 %       | 600 s |
| TC_Line  | 8        | 62.5 s    | 99 %         | 600 s |
| TC_Oven  | 10       | 62.5 s    | 99.5 %       | 1800 s |
| Q_Station| 1        | 62.5 s    | 99 %         | 600 s |
| CC_HRK   | 1        | 62.5 s    | 98 %         | 600 s |
| SR       | 1        | **? (you must find it — Prio 2)** | 99.9 % | 600 s |
| MR       | 1        | **?**     | 99.9 %       | 600 s |
| ReRun    | 1        | **?**     | 99.9 %       | 600 s |

**Buffers:** every buffer has a time of 10 s per place. Bu_6 … Bu_11 have **fixed capacity 1**. Bu_1–Bu_5, Bu_12 and the Rework Buffer have capacity **? (you must minimize — Prio 3)**.

**Rework probabilities at the Q-Station (per color)**

| Color | SR  | MR  | ReRun | OK  |
|-------|-----|-----|-------|-----|
| red   | 7 % | 2 % | 1 %   | 90 % |
| white | 9 % | 6 % | 5 %   | 80 % |
| black | 8 % | 3 % | 2 %   | 87 % |

**Shift model** (applies to TC_Entry, Q_Station, CC_HRK, SR, MR, ReRun — *not* to TC_Line and TC_Oven, which run continuously):

- Shift 1: 06:00–14:00, Mo–Fr, pauses 9:00–9:15 and 12:00–12:30
- Shift 2: 14:10–22:10, Mo–Fr, pauses 18:10–18:40 and 20:30–20:45

(The Excel values 0.25 / 0.58333 / 0.59028 / 0.92361 are day fractions: 0.25 × 24 h = 06:00, etc.)

That gives **14.5 productive hours per day** (2 × 8 h minus 2 × 45 min pauses), so your daily target is **53 × 14.5 ≈ 769 parts/day** at the Ship, or ~3,843 parts per 5-day week. Check this number at the Drain after every run.

---

## 1. Create the model and learn the screen

1. Start **Tecnomatix Plant Simulation 2404**.
2. On the start page click **Create New Model**.
3. A dialog asks about 2D/3D. Choose **2D only** (you can ignore 3D for this task) and confirm with **OK**.
4. You now see:
   - **Class Library** (tree on the left): contains all object templates (`MaterialFlow`, `Resources`, `InformationFlow`, `MUs`, `Tools`…).
   - **Toolbox** (strip of icons above the working area, with tabs *Material Flow*, *Resources*, *Information Flow*, *User Interface*, *MUs*, *Tools*): the same objects as drag-and-drop icons.
   - **Frame window** (big empty area in the middle, usually with an **EventController** already placed in the corner): this is where you build.
   - **Console** (bottom): error messages appear here. Read it whenever something doesn't work.
5. Save immediately: **File ▸ Save As**, give it a name like `Rework_PaintShop_G5.spp`. Save often (Ctrl+S).

> **Naming rule:** object names may not contain spaces, `-` or `/`. That's why we write `Q_Station` and `CC_HRK`.

---

## 2. Create the three part types (red, white, black)

The parts ("MUs" = Moving Units) live in the Class Library, not in the frame.

1. In the **Class Library**, open the folder **MUs**. You'll see **Entity** (a simple part).
2. Right-click **Entity** ▸ **Duplicate**. Rename the copy (right-click ▸ **Rename**, or F2) to **Part**.
3. Give *Part* a custom attribute that will store the quality result:
   1. Double-click **Part** to open it.
   2. Go to the **User-defined** tab.
   3. Right-click in the empty list ▸ **New** (or click the "new attribute" icon).
   4. Name: `QualityState` — Data type: `string` — Value: `OK`. Click **OK**, close the dialog.
4. Now create the three colors as children of *Part*: right-click **Part** ▸ **Derive**. Do this three times and rename the children **Part_red**, **Part_white**, **Part_black**. Because they are *derived* from Part, all three automatically have the `QualityState` attribute.
5. (Optional cosmetics: open a class, ribbon ▸ icon editor, recolor the little box red/white/black. The model works fine without this.)

---

## 3. Drag the layout into the frame

### 3.1 Which object for what?

| You need | Object in Toolbox tab **Material Flow** | Why |
|---|---|---|
| Order (part creation) | **Source** | creates MUs |
| Bu_1 … Bu_12 | **Buffer** | FIFO queue with capacity + dwell time |
| TC_Entry, Q_Station, CC_HRK, SR, MR, ReRun | **Station** | capacity 1, one processing time |
| TC_Line, TC_Oven | **ParallelStation** | capacity > 1. Capacity 8 × takt 62.5 s ⇒ each part stays 8 × 62.5 = **500 s** in the line, and one part leaves every 62.5 s — exactly the real behavior. Oven: 10 × 62.5 = **625 s**. |
| Rework Buffer (random access) | **Store** | parts placed in a Store **never move on by themselves** — a Method takes any part out at any time. That is literally "random access" (and "Object Store" is one of the options your task sheet explicitly allows). |
| Ship (exit) | **Drain** | destroys MUs and counts throughput |
| Routing logic | **Method** (Toolbox tab **Information Flow**) | small SimTalk programs |
| Shifts | **ShiftCalendar** (Toolbox tab **Resources**) | working times + pauses |

### 3.2 Place and rename everything

Drag each object from the Toolbox into the frame (click the icon, then click in the frame). To rename: double-click the object and edit the **Name** field at the top of its dialog, or select it and press F2.

Place these **23 objects** (suggested arrangement — main line left→right across the top, rework area below, like Figure 1 of your task sheet):

- Top row: `Order` (Source), `Bu_1`, `Bu_2`, `TC_Entry`, `TC_Line` (ParallelStation), `TC_Oven` (ParallelStation), `Bu_3`, `Q_Station`, `Bu_4`, `Bu_5`, `CC_HRK`, `Ship` (Drain)
- Middle: `Rework_Store` (Store)
- Rework rows: `Bu_6` → `SR` → `Bu_9`;  `Bu_7` → `MR` → `Bu_10`;  `Bu_8` → `ReRun` → `Bu_11`
- Return path: `Bu_12` (below the main line, pointing back to Bu_2)
- Logic: two Methods named `QualityCheck` and `DispatchRework`; one `ShiftCalendar`
- The `EventController` is already there.

### 3.3 Connect with the Connector tool — exact order

In the Toolbox (*Material Flow* tab) **double-click** the **Connector** icon — double-clicking keeps the tool active so you can draw many connections in a row (press **Esc** when done). For each connection: click the *from* object, then the *to* object. The arrow direction = flow direction, so the click order matters!

Draw exactly these connections:

1. `Order → Bu_1`
2. `Bu_1 → Bu_2`
3. `Bu_2 → TC_Entry`
4. `TC_Entry → TC_Line`
5. `TC_Line → TC_Oven`
6. `TC_Oven → Bu_3`
7. `Bu_3 → Q_Station`
8. `Q_Station → Bu_4`  *(the OK path — actual routing is done by a Method in step 5, but draw it for clarity)*
9. `Q_Station → Rework_Store`  *(the NOK path)*
10. `Bu_4 → Bu_5`
11. `Bu_5 → CC_HRK`
12. `CC_HRK → Ship`
13. `Bu_6 → SR`
14. `SR → Bu_9`
15. `Bu_9 → Bu_5`  *(repaired SR parts merge back before cavity conservation)*
16. `Bu_7 → MR`
17. `MR → Bu_10`
18. `Bu_10 → Bu_5`
19. `Bu_8 → ReRun`
20. `ReRun → Bu_11`
21. `Bu_11 → Bu_12`
22. `Bu_12 → Bu_2`  *(the ReRun loop back into the topcoat line)*

**Do NOT draw connectors from `Rework_Store` to Bu_6/Bu_7/Bu_8.** The Store is passive on purpose — our `DispatchRework` method will pull the right part to the right lane (step 6). Leaving the connectors out makes it impossible for parts to "escape" the wrong way.

---

## 4. Enter the Excel data

### 4.1 The Source ("Order")

1. **Double-click `Order`.**
2. **MU selection:** choose **Random** from the dropdown.
3. A **Table** field appears — click its **Open**/`...` button. Fill three rows:
   - Row 1: MU = `Part_red` (easiest: drag the class **Part_red** from the Class Library straight into the cell, or type `.MUs.Part_red`), relative frequency = `15`
   - Row 2: `Part_white`, `60`
   - Row 3: `Part_black`, `25`
   Close the table.
4. **Time of creation:** *Interval Adjustable*. Set **Interval = 0** (a `0` means: create a new part the moment there is space in Bu_1 — the source never starves the line, which is what you want for a throughput study). Leave **Amount** at `-1` (unlimited).
5. **OK** to close.

### 4.2 Processing time of a machine (your Question 2a)

For **every Station/ParallelStation**: **double-click the object ▸ tab "Times" ▸ field "Processing time"**. Leave the dropdown on **Const** and type the value in the box next to it. You can simply type seconds (`62.5`) — Plant Simulation converts it to `1:02.5000` (the display format is mm:ss).

| Object | Processing time to type |
|---|---|
| TC_Entry | `62` |
| TC_Line | `500`  *(= 8 × 62.5)* |
| TC_Oven | `625`  *(= 10 × 62.5)* |
| Q_Station | `62.5` |
| CC_HRK | `62.5` |
| SR / MR / ReRun | start values from step 8, e.g. `700` / `1270` / `1620` — these are your Prio-2 experiment knobs |

**Capacity of the ParallelStations:** double-click `TC_Line` ▸ tab **Attributes** ▸ set **X-dimension = 8**, Y-dimension = 1. For `TC_Oven`: X-dimension = **10**. (Capacity = X × Y.)

### 4.3 Capacity and time of a buffer (your Question 2b)

For **every Buffer Bu_1 … Bu_12**: **double-click ▸ tab "Attributes" ▸ field "Capacity"**. Set **Buffer type = Queue** (FIFO). Then on the **Times** tab set the **Processing time / Dwell time** to `10` (the "10 s each place" from Excel).

- `Bu_6, Bu_7, Bu_8, Bu_9, Bu_10, Bu_11`: Capacity = **1** (fixed by Excel).
- `Bu_1, Bu_2, Bu_3, Bu_4, Bu_5, Bu_12`: these are the **?** buffers. **Start generously, e.g. Capacity = 50**, get the model running and hitting 769/day, *then* shrink them step by step (Prio 3).

**Rework_Store:** double-click ▸ **Attributes** ▸ **X-dimension / Y-dimension** define its capacity (e.g. X = 10, Y = 5 ⇒ 50 places to start; shrink later — it is part of Prio 3).

### 4.4 Failures (availability + MTTR)

For each of the 8 stations: **double-click ▸ tab "Failures" ▸ button "New…"**. In the failure dialog:

1. Give it a name (e.g. `Breakdown`), make sure **Active** is checked.
2. Choose the entry mode **Availability + MTTR**.
3. Type the values from the table in section 0 — e.g. TC_Entry: Availability `98.5`, MTTR `10:00` (= 600 s). TC_Oven gets MTTR `30:00` (= 1800 s).
4. For "failure relates to", pick **Simulation time** (simplest consistent choice — mention it in your report).
5. **OK**, **OK**.

### 4.5 Shift calendar

1. **Double-click your `ShiftCalendar`** object. You get a table of shifts.
2. Row 1: Name `Shift1`, From `6:00`, To `14:00`, tick **Mo Tu We Th Fr**, Pauses: `9:00-9:15; 12:00-12:30`.
3. Row 2: Name `Shift2`, From `14:10`, To `22:10`, tick **Mo–Fr**, Pauses: `18:10-18:40; 20:30-20:45`.
   *(Depending on the build, the Pauses cell is either typed directly like this or opens a small sub-table when you click into it — enter both pause windows either way. Press F1 in the dialog if unsure.)*
4. Assign it to the six stations listed in Excel — `TC_Entry`, `Q_Station`, `CC_HRK`, `SR`, `MR`, `ReRun`: **double-click the station ▸ tab "Controls" ▸ field "Shift calendar" ▸ select your ShiftCalendar**. Do **not** assign it to TC_Line, TC_Oven or any buffer.

---

## 5. The Quality Routing (your Question 3)

Plant Simulation's built-in "Percentage" exit strategy can't do *different* percentages per color, so we use a tiny SimTalk program as the station's **exit control**. It runs once for every part that finishes at the Q-Station: it rolls a random number, compares it with that color's probabilities, writes the result into the part's `QualityState` attribute, and moves the part.

1. **Double-click the Method `QualityCheck`** (the editor opens) and paste exactly this:

```
-- Exit control of Q_Station:
-- decides the quality state per color and routes the part.
-- @ is the part that is currently on the Q_Station.

var pSR, pMR, pRR: real

if @.name = "Part_red"
	pSR := 0.07
	pMR := 0.02
	pRR := 0.01
elseif @.name = "Part_white"
	pSR := 0.09
	pMR := 0.06
	pRR := 0.05
else -- Part_black
	pSR := 0.08
	pMR := 0.03
	pRR := 0.02
end

var r: real := z_uniform(1, 0, 1)

if r < pSR
	@.QualityState := "SR"
elseif r < pSR + pMR
	@.QualityState := "MR"
elseif r < pSR + pMR + pRR
	@.QualityState := "ReRun"
else
	@.QualityState := "OK"
end

if @.QualityState = "OK"
	waituntil not Bu_4.full prio 1
	@.move(Bu_4)
else
	waituntil not Rework_Store.full prio 1
	@.move(Rework_Store)
end
```

How it works: `z_uniform(1,0,1)` draws a random number between 0 and 1 (stream 1). For a white part, `r < 0.09` ⇒ SR; `r < 0.15` ⇒ MR; `r < 0.20` ⇒ ReRun; otherwise OK — i.e. exactly 9/6/5/80 %. The `waituntil … prio 1` lines make the part *wait on the Q-Station* if the target is momentarily full and move the instant space frees up (that is realistic blocking and prevents parts from getting stuck forever).

2. **Hook it up:** double-click `Q_Station` ▸ tab **Controls** ▸ field **Exit** ▸ select `QualityCheck`. (As soon as an exit control is set, the station ignores its connectors and does only what the method says.)
3. Press **F7** in the method editor (or the check-mark icon) to compile — any typo shows up in the Console.

> Note for ReRun parts: when they come around the loop and reach the Q-Station a second time, the method simply rolls the dice again — which is exactly what the task intends (a re-painted part can fail again).

---

## 6. The random-access Rework Store

Parts sit in `Rework_Store` and wait. A dispatcher method scans the store and pulls **any** part whose lane is free — SR parts to Bu_6, MR to Bu_7, ReRun to Bu_8. We trigger it at the only two moments anything can change: when a new part *arrives* in the store, and when a rework station *takes* a part (which frees its capacity-1 buffer).

1. **Double-click `DispatchRework`** and paste:

```
-- Pulls matching parts out of the Rework_Store (random access).
-- Called: after a part enters Rework_Store,
--         and after a part enters SR, MR or ReRun.

for var i := Rework_Store.numMU downto 1
	var part: object := Rework_Store.MU(i)
	if part.QualityState = "SR" and Bu_6.empty
		part.move(Bu_6)
	elseif part.QualityState = "MR" and Bu_7.empty
		part.move(Bu_7)
	elseif part.QualityState = "ReRun" and Bu_8.empty
		part.move(Bu_8)
	end
next
```

(We loop **backwards** because moving a part out of the store shifts the indices of the parts behind it.)

2. **Hook it up in four places** (tab **Controls** ▸ field **Entrance** ▸ select `DispatchRework`; leave any "front" checkbox unticked so it runs *after* the part has entered):
   - on `Rework_Store`
   - on `SR`
   - on `MR`
   - on `ReRun`

That's the whole rework logic. Bu_6/7/8 (capacity 1, 10 s) feed the stations automatically via their connectors.

---

## 7. Run it

1. **Double-click the EventController.** Set **End** to e.g. `35:00:00:00` (= 35 days ≈ 5 weeks; format is days:hours:minutes:seconds).
2. Click **Reset** (the model empties, statistics clear), then **Start**. Use the **fast-forward** button (or drag the speed slider to max / enable "max speed") so weeks pass in seconds.
3. **Read your result:** double-click `Ship` ▸ tab **Statistics**. You'll see total throughput and throughput per day/hour. **Target: ≥ 769 parts per day** (≈ 3,843 per 5-day week). Careful: "per hour" in the statistics is per *calendar* hour (24 h/day, 7 d/week), while your 53 uph target refers to *working* hours — comparing per **day** avoids that confusion.
4. **Find bottlenecks:** open any station ▸ tab **Statistics** — the share of *Working / Blocked / Waiting / Failed / Paused* tells you everything. A station that is heavily **blocked** needs more buffer *behind* it; one that is **waiting** is starved from the front.
5. Watch the model animate at slow speed for a few minutes once: do parts actually flow into the rework lanes? Does a ReRun part travel Bu_11 → Bu_12 → Bu_2 and re-enter the line? Does the Console stay clean?

**Warm-up / steady state (a "useful question" from your task sheet):** the model starts empty, so the first hours produce too little — that biases the statistics. Show in your report that you considered it: e.g. plot throughput per day over the run; once the daily value is stable, the system is in steady state. Practical approach: simulate ≥ 4–5 weeks so the empty first day barely matters, and/or exclude day 1 from your evaluation. Use the **ExperimentManager** (Toolbox ▸ Tools) later to run several replications per parameter setting for the report.

---

## 8. The three priorities — how to experiment

**Prio 1 — hit 769/day.** Quick feasibility check (good content for your report): the line must deliver 53 OK-parts per working hour, but ~3.65 % of all inspected parts are ReRuns that go around again (0.15·1 % + 0.60·5 % + 0.25·2 %), so the topcoat line must handle ≈ 53 / 0.9635 ≈ **55 parts gross per hour**. Its takt of 62.5 s allows 57.6/h, minus failures ⇒ ~56–57/h. It fits, but it's tight — which is exactly why buffer sizes and rework takts matter.

**Prio 2 — maximum cycle times of SR / MR / ReRun.** Per gross hour the rework arrival rates are roughly:

- SR: 55 × 8.45 % ≈ 4.65/h ⇒ theoretical maximum ≈ 3600/4.65 ≈ **~775 s**
- MR: 55 × 4.65 % ≈ 2.56/h ⇒ ≈ **~1405 s**
- ReRun: 55 × 3.65 % ≈ 2.01/h ⇒ ≈ **~1790 s**

(8.45 % = 0.15·7 % + 0.60·9 % + 0.25·8 %, etc.) These are *upper bounds*; randomness eats some margin. Start ~10 % lower (e.g. **700 / 1270 / 1620 s**), confirm the target is met and the Rework_Store doesn't keep growing, then raise each takt in steps (e.g. +25 s) until the throughput target just breaks — the last good value is your answer. Change one station at a time and document every run in a table (run no., SR/MR/ReRun takt, buffer sizes, parts/day) — that table goes straight into your report.

**Prio 3 — minimize the ? buffers and the Rework_Store.** Only after Prio 1 and 2 are settled. Reduce one buffer at a time (50 → 20 → 10 → 5 → …) and re-run; keep the smallest capacity at which you still hit 769/day. Expect Bu_3 (between oven and Q-Station) and Bu_12 to matter for decoupling failures, while Bu_1 can usually be tiny (the Source refills it instantly). Watch one danger: if the Rework_Store is made too small *and* fills up with ReRun parts while the line is full, the loop can gridlock — if you ever see throughput collapse to zero, check exactly that and size the store/Bu_12 up a notch.

---

## 9. Troubleshooting

- **Nothing moves after Start:** did you press **Reset** first? Is the Source's interval really 0 and MU table filled?
- **Yellow/red message in the Console:** double-click the message — it jumps to the faulty method line. Typical: a typo in an object name (names are case-sensitive in the sense that they must match exactly).
- **Parts pile up on the Q-Station forever:** the Exit control isn't assigned, or `Bu_4` / `Rework_Store` names in the method don't match your objects.
- **Rework lanes stay empty:** `DispatchRework` not assigned as *Entrance* control on all four objects (Store + 3 stations).
- **`.full` not accepted in `waituntil`** (rare, build-dependent): replace `not Bu_4.full` with `Bu_4.numMU < Bu_4.capacity` (same for the store).
- **Throughput is ~0 at night/weekends:** correct! Six stations follow the shift calendar. Judge throughput per *day*, not per calendar hour.
- **Old object names in course slides:** *Station* was called *SingleProc* and *ParallelStation* was called *ParallelProc* in older versions — same objects.

## 10. Two things to confirm with your supervisor (write the assumption in your report either way)

1. **"53 uph"** — per *working* hour (⇒ 769/day with the 14.5 h shift model; this guide assumes that) or per calendar hour (⇒ 1,272/day, which the line physically cannot do — so working hours is almost certainly meant).
2. **"10 s each place"** for buffers — this guide models it as a 10 s dwell time per part in each buffer (the usual course interpretation). If your instructor means a conveyor where a part needs 10 s *per place* it travels through, the buffers would be modeled as *Line* objects instead; ask before the final report.

## 11. For the report (VDI 3633 mapping, in one breath)

Task definition = section 0 here; system analysis/conceptual model = Figure 1 + the object table in 3.1; data collection = the Excel tables; implementation = steps 2–6 (describe `QualityCheck` and `DispatchRework` — these are your "techniques beyond the basics"); verification = step 7.5 (animation check, Console clean, plausibility: ~20 % of white parts go to rework); validation = the feasibility math in step 8; experiments = your run table for Prio 1–3; documentation = the report itself. Parameters that change between runs: SR/MR/ReRun processing times and the capacities of Bu_1–5, Bu_12 and Rework_Store.

Good luck — and run the model after *every* small change. Errors are easy to find when only one thing changed.
