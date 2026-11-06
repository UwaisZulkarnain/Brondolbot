# BrondolBot

Autonomous palm oil loose-fruit (brondolan) collector, simulated in Webots.
SAIA 3383 Robotics, Semester 2026/2027-1, Group 6, Track P1 (Autonomous Mobile Robot).

**Status:** planning (Week 5). The structure below is the agreed target. Folders appear as each module lands.

---

## What it does

- A Pioneer 3-AT drives the paths between palm rows in a scaled plantation block
- It finds loose fruit with a camera, works out the shortest collection route, and avoids trunks, frond piles and workers
- When the fruit list is empty it returns to the collection point
- A fixed-sweep baseline runs on the same robot and wheel controller, so the AI and non-AI results compare fairly

## AI layers

| Layer | Method | Input | Output |
|---|---|---|---|
| Perception | YOLOv8n object detector, trained on Webots images | RGB camera frame | Fruit boxes + confidence |
| Planning | Occupancy grid + A* + nearest-neighbour/2-opt visit order | Grid, pose, fruit list | Ordered targets + collision-free path |

## Repository layout

```
brondolbot/
├── worlds/
│   └── plantation.wbt            # field, palms, robot, sensors
├── protos/                       # palm, fruit cluster, frond pile (if custom)
├── controllers/
│   ├── brondolbot/               # robot controller, AI and baseline modes
│   │   ├── brondolbot.py         # main loop + state machine
│   │   ├── perception.py         # detector wrapper, pixel -> robot frame -> map
│   │   ├── planner.py            # occupancy grid, A*, visit order, replanning
│   │   ├── motion.py             # heading controller, inverse kinematics
│   │   ├── baseline.py           # fixed sweep waypoints
│   │   └── config.py             # robot constants and thresholds, one place only
│   └── field_supervisor/
│       └── field_supervisor.py   # spawns fruit by seed, scores the run, writes CSV
├── detector/
│   ├── capture_dataset.py        # saves images + auto labels from Camera Recognition
│   ├── train_yolov8n.ipynb       # training notebook (Colab)
│   └── weights/brondol_fruit.onnx
├── tests/
│   ├── scenarios/                # S1.json, S2.json, S3.json
│   ├── run_batch.py              # runs every scenario x seed x mode headless
│   └── test_planner.py           # unit tests for A* and visit order
├── results/                      # one CSV per batch, charts
├── docs/
│   ├── logs/                     # one contribution log per member
│   └── figures/
├── requirements.txt
└── README.md
```

## Setup

Requirements: Webots R2025a, Python 3.10 or newer, Git.

```bash
git clone <repo-url>
cd brondolbot
python -m pip install -r requirements.txt
```

In Webots: **Tools > Preferences > General > Python command**, set it to the same Python you installed the requirements into. If Webots uses a different Python, the controller will fail on import.

Detector inference runs from the exported ONNX file with `onnxruntime`, so nobody needs PyTorch locally. Training happens in Colab only.

## Running

**One run (for development)**

1. Open `worlds/plantation.wbt` in Webots
2. The Supervisor reads `tests/current_run.json` for scenario, seed and mode (default: S1, seed 1, AI)
3. Press play

**Full test batch (30 runs)**

```bash
python tests/run_batch.py --scenarios S1 S2 S3 --seeds 5 --modes ai baseline
```

For each run, the script writes `tests/current_run.json`, launches Webots with `--batch --mode=fast --minimize`, and waits. The Supervisor ends the simulation when the run finishes or hits 600 s, then appends one row to `results/`.

Check in Week 10 whether `--no-rendering` still lets the robot camera update. Only add that flag if it does.

## Interfaces

Agreed in Week 6 so everyone can build in parallel. Change one only after telling the people who use it.

| Interface | Owner | Format |
|---|---|---|
| Pose | Farkhin | `(x, y, theta)` in metres and radians, theta counter-clockwise from +x, in [-pi, pi] |
| LiDAR | Farkhin | `lidar.getRangeImage()`, metres; FOV and resolution in `config.py` |
| `detect(image)` | Irfan Johan | `list[(u, v, w, h, conf)]`, box centre and size in pixels |
| `fruit_to_map(det, depth, pose)` | Uwais | `(x, y)` in map metres |
| `plan(grid, pose, goal)` | Haziem | `list[(x, y)]` waypoints, or `None` if unreachable |
| `visit_order(pose, targets)` | Haziem | targets reordered |
| Robot to Supervisor | Uwais | robot writes `{"replans": n, "state": "..."}` to its `customData`; the Supervisor reads it at the end of the run |
| Results CSV | Farkhin, used by Hajar | `run_id, scenario, seed, mode, fruit_placed, fruit_collected, collection_rate, goal_reached, time_s, path_length_m, collisions, replans` |

Rule: the robot controller never reads ground truth from the Supervisor. It only uses its own sensors.

## Team roles and scope

| Member | Role | Owns (files) | Done when |
|---|---|---|---|
| **Muhd Uwais Zulkarnain** (leader) | System architect, integrator | `brondolbot.py`, `perception.py` (fruit-to-map part), Robot-Supervisor link | All six states run end to end on S1 seed 1. Mapped fruit lands within 0.3 m of its true spot. `--mode` switch works. Every PR reviewed and merged. |
| **Muhammad Haziem Azfar** | AI planning lead | `planner.py`, `test_planner.py` | A* passes unit tests (shortest path, blocked goal returns `None`). Replanning on a 160 x 160 grid finishes inside one control step. 2-opt order is never longer than nearest neighbour. |
| **Muhammad Farkhin** | Simulation and sensors lead | `plantation.wbt`, `protos/`, `field_supervisor.py`, `config.py` (robot constants) | World loads with no warnings. Every sensor returns data. Same seed gives the same fruit positions every time. Fruit is removed within 0.3 m. Collision count is checked by driving into a trunk. One CSV row per run. |
| **Ahmad Irfan Johan** | Perception lead | `detector/`, `perception.py` (`detect()` part) | At least 1,000 auto-labelled images, split 70/15/15. mAP@0.5 of at least 0.85 on the test split. ONNX exported. `detect()` runs in 150 ms or less per frame on a laptop CPU. |
| **Wan Nur Hajar** | Testing and analysis lead | `tests/scenarios/`, `run_batch.py`, `baseline.py`, `results/` | S1 to S3 configs for seeds 1 to 15. Baseline sweep route works. All 30 runs logged. Results table (mean and standard deviation) plus charts. |
| **Amirah Yasmine Fathiah** | Report and media lead | `README.md`, `docs/`, demo video | README matches the code at submission. Report formatted (Times New Roman 10 pt, 1.5 spacing). 3 to 5 minute video shows all three scenarios, AI vs baseline. Contributions section built from member logs. |

Everyone also writes their own report sections (see the proposal doc) and keeps `docs/logs/<name>.md` up to date: date, what you did, commit links. Those logs become the Part B contribution statements.

### Project scope

**In scope**
- One robot, one 16 m x 16 m plantation block, 20 palms (4 rows x 5, 3 m spacing)
- 8 to 12 fruit clusters per run, placed by seed
- Static obstacles: frond piles, a standing worker
- Pose from GPS + compass. Fruit from camera + depth. Obstacles from LiDAR.
- Collection simulated: a fruit counts when the robot comes within 0.3 m of it
- 3 scenarios x 5 seeds x 2 modes = 30 scored runs

**Out of scope**
- Real hardware and the picking mechanism itself
- Harvesting bunches from the palm
- Mud, slopes, weather, night
- Multiple robots
- Moving workers (stretch goal, only after Week 10)

### Scenarios

| | Condition | Seeds | Target |
|---|---|---|---|
| S1 | Clear paths, 10 fruit clusters | 1 to 5 | At least 90% collected, back home, 0 collisions |
| S2 | S1 plus 2 frond piles and 1 standing worker | 6 to 10 | At least 90% collected, 0 collisions, replans around blocks |
| S3 | New fruit layout, start and collection point moved | 11 to 15 | At least 85% collected, 0 collisions, no code changes |

## Git workflow

- `main` must always run. No direct pushes to `main`.
- One branch per module: `feat/world`, `feat/detector`, `feat/planner`, `feat/integration`, `feat/testing`, `docs/report`
- Open a pull request into `main`. Uwais reviews and merges.
- Pull `main` before you start work, and merge `main` into your branch before opening a PR
- Short, clear commit messages: `Add A* with 8-connected moves`, not `update`
- Any change to a shared interface: say so in the group chat first

**Never commit**
- Dataset images and training checkpoints (`.pt`): keep them in the shared Google Drive. Only the final `.onnx` goes in the repo.
- Videos: upload to Drive and put the link in `docs/`
- `__pycache__/`, `.venv/`, Webots cache and OS files (covered by `.gitignore`)

```gitignore
__pycache__/
*.pyc
.venv/
*.pt
detector/dataset/
detector/runs/
*.mp4
.DS_Store
Thumbs.db
```

## Timeline

| Week | Milestone |
|---|---|
| 6 | Interfaces locked; world, detector and planner work starts |
| 8 | Part A drafts in; Uwais merges and the team reviews |
| 9 | **Part A proposal submitted**; detector and planner done |
| 10 | Full system runs end to end |
| 11 | All 30 runs done, video recorded, report drafts in |
| 12 | **Part B report and project folder submitted** |

## Submission checklist

**Part A, Week 9**
- [ ] Proposal PDF, max 10 pages, Times New Roman 10 pt, 1.5 spacing
- [ ] Six roles named

**Part B, Week 12**
- [ ] Report PDF, max 25 pages
- [ ] Webots project folder: `.wbt`, controllers, protos, plugins if any
- [ ] Test data (`results/` CSVs)
- [ ] 3 to 5 minute demonstration video
- [ ] Individual contribution statements for all six members
