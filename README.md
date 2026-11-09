# Machine Learning Assisted-Genetic Algorithm
## Introduction
This project aims to provide a flexible automated solution for designing dual band (may be easily extended by altering the fitness calculation method) planar antennas. The whole optimization process is automated by utilizing Python scripts to control CST Studio Suite® 2025 (2023 is also valid). Genetic algorithm (BPSO is another possible method and may be adopted in the future) is used to optimize the antenna topology. The antenna topogy will gradually evolve from a arbitrary design to a spec-satisfying design. The design region of the antenna topology is a pixelated rectangular area with user-defined length, width, height, fineness, and position. This project can be easily integrated with any CST simulation files by directly drawing the design region at the user-defined location in the files. Machine learning-based surrogate model can be used to accelerate the genetic algortihm optimization process by roughly 10~15% faster in average (optional). 

---
## Known Issues and Caveats
This code is related to Chapter 4 of my master's thesis (NTU, 2025, doi:10.6342/NTU202501972). A review in Nov 2026 found the issues below. They have **not** been fixed yet. Please read them before reusing the code or citing the results.

**Reproducibility**
- No version of this repository contains the exact Chapter 4 setup (1.5 GHz, 36 mm x 36 mm region, 4 x 4 pixels with the central 4 prefilled). The current settings are for a dual-band 15 x 6 region.
- There is no script for the 50-trial average in thesis Fig. 4.17.

**GA bugs**
- `MUTATE_RATE = 1/POPULATION_SIZE*CONVERGENCE*30` is far too large: 0.6 in the June 2025 version and about 1.02 with the current `settings.json`. At 1.02 *every* bit of every child is flipped. The GA then behaves close to random search with elitism, which weakens the GA vs. MLAO-GA comparison. A common choice is about 1/(number of pixels).
- The loop condition `while (best_fitness < CONVERGENCE) or (len(...) < MAX_ITERATIONS)` should use `and`. With `or`, the loop never stops if the design never converges.
- The initial population uses `np.random.randint(pixels, size=POPULATION_SIZE)`, i.e. integers in [0, NX*NY). This only switches on the lowest-index pixels. It should be `np.random.randint(0, 2, size=(POPULATION_SIZE, NX*NY))`.
- Selection keeps the top sqrt(N) individuals (truncation selection). The thesis calls it tournament selection.
- "Iterations" count cached (already simulated) designs too, so they are not the number of EM simulations. The cache in `fitness_table.csv` is also shared across runs.

**MLAO-GA bugs**
- Data augmentation uses `individual[::-1]`, which rotates the grid by 180 degrees. With an off-centre feed, the rotated design is a different antenna, so it gets a wrong fitness label. The valid symmetry for a feed on y = 0 is a mirror across y = 0 (flip the rows only).
- `AntennaCNN` reshapes the input as `(NX, NY)`, but pixels are laid out as `(NY, NX)` (see `set_domain` and `plot.py`). For non-square grids the CNN sees scrambled neighbours.
- In the plain-CST branch, `best_fitness` is recorded *before* it is updated. The progression therefore lags one generation, and generation 0 records -1000.
- `avg_last = 0` is reset at the start of every generation, so the check `pop_avg_fitness < avg_last` never fires.
- After a surrogate-only generation, `best_fitness` holds a *predicted* value, which the loop condition can use to stop without a CST check.
- The surrogate is used once its validation MSE is below 0.2 on min-max scaled fitness. That is a loose threshold (RMSE about 0.45 of the range). The validation split also contains mirrored copies of training samples, so the MSE is optimistic.

---
## Setup
Before running the code, ensure you have the following installed:
- **CST Studio Suite®**: To solve EM problems. Official interface only support python 3.6~3.9.
- **Git**: To clone this repository.
- **Miniconda** or **Anaconda**: To manage the Python environment and dependencies.

### 1. Install Git
- **Windows**: Download and install Git from [git-scm.com](https://git-scm.com/downloads). Follow the default installation options.

Verify Git is installed by running:
```
git --version
```

### 2. Install Miniconda
Miniconda is a lightweight version of Anaconda that manages Python environments.
- Download Miniconda from [docs.conda.io](https://docs.conda.io/en/latest/miniconda.html) (choose the version for your OS: Windows, Mac, or Linux).
- Follow the installer instructions:
  - **Windows**: Double-click the `.exe` file and follow the prompts. Check "Add Miniconda to my PATH" if available.
- Restart your terminal after installation.

Verify Miniconda is installed by running:
```
conda --version
```

### 3. Clone the Repository
In a terminal or command prompt, navigate to the folder where you want the project and run:
```
git clone https://github.com/electronics10/MLAO2.git
```
Then, enter the project directory:
```
cd MLAO2
```

### 4. Set Up the Conda Environment
All dependencies can be find in two of my other projects [[1]](https://github.com/electronics10/Topology_Optimization) and [[2]](https://github.com/electronics10/mlpifa).

After dependencies are installed:
- Activate the environment:
```
conda activate my_environment_name
```

### 5. Run the Code
1. Users can change the settings in `settings.json` for tailored use (e.g., S11 goal, design region size, hyperparameters).
2. Initialize the pixelated design region in a ".cst" file (automatically create `./CST_Antennas/topop.cst` if not specified). 
```
python initialize.py
```
3. Excitation port and other setup (e.g., GPU acceleration) in the ".cst" file should be defined by the user before optimization.
4. Design the antenna by running optimization code.
  ```
  python GA.py
  ```
  or
  ```
  python MLAOGA.py
  ```
  for ML usage. Random seed is assigned (2 in default) for replication in case CST crashes (S11 files would be saved to `./s11` for each antenna topology simulation). General optimization process info will be saved to `./data/log_seed.txt` and `./data/GA_seed.csv`. The final design will be plot after optimization.

5. Arbritray antenna topologies (solved in the process) and their corresponding S11 can be plot by running:
```
python plot.py
```
6. One can load arbritrary antenna topology into CST for later usage.
```
python verify.py
```
