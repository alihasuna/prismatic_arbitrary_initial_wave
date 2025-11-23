# Using arbitrary incoming waves with prismatique

The updated HRTEM interpolation code accepts continuous tilt vectors and records
per-tilt beam weights so you can drive Prismatic with arbitrary incident waves.
This walkthrough shows how to wire those tilts into prismatique end to end.

## 1. Build Prismatic locally
Compile the CLI binary so prismatique can launch the updated executable:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target prism
```

## 2. Generate a tilt list (arbitrary wavefront)
Create the list of tilt vectors that describe your wavefront. Each entry is
`(x_tilt_mrad, y_tilt_mrad)`. The C++ side converts to radians and computes
neighboring Fourier beams automatically.

Example Python snippet that writes a simple CSV tilt table:

```python
import numpy as np

tilt_grid = np.stack(
    np.meshgrid(np.linspace(-15, 15, 5), np.linspace(-10, 10, 3), indexing="ij"),
    axis=-1,
).reshape(-1, 2)
np.savetxt("tilts.csv", tilt_grid, delimiter=",")
```

## 3. Teach your wrapper to emit the tilts
When your Python wrapper prepares Prismatic metadata, load the tilt table and
store it in the HRTEM tilt arrays that are already persisted to disk. The C++
`Parameters` structure reads `xTilts_tem` and `yTilts_tem` and now builds
`hrtemInterpolationIndices/Weights` for each requested tilt, so no grid snapping
is required.

Example (abbreviated) wrapper hook:

```python
from pyprismatic import params
import numpy as np

p = params.Params()
# ... fill other fields ...
tilts = np.loadtxt("tilts.csv", delimiter=",")
p.xTilts_tem = tilts[:, 0].tolist()  # mrad
p.yTilts_tem = tilts[:, 1].tolist()  # mrad
p.write("hrtem_params.txt")
```

## 4. Wire prismatique to the new metadata
Use a prismatique job to run the locally built binary while pointing to the
tilt-aware parameter file. The job can also run the tilt-generation script as a
`pre` step if you want a single command.

```yaml
working_directory: prismatique-output
runs:
  - name: hrtem-arbitrary-wave
    pre:
      - python scripts/make_tilts.py      # generates tilts.csv
      - python scripts/write_params.py    # writes hrtem_params.txt
    command: >
      ${PRISM_BIN:-./build/bin/prism}
      --input-file SI100.XYZ
      --output-file prismatique-output/hrtem_arbitrary_wave.h5
      --algorithm p
      --interp-factor 8
      --pixel-size 0.05
      --cell-dimension 20 20 20
      --energy 300
      --alpha-max 25
    env:
      PRISMATIC_PARAM_FILE: hrtem_params.txt
```

## 5. Inspect the results
After `prismatique run`, the generated HDF5 output contains one datacube entry
per requested tilt. The interpolation metadata allows Prismatic to combine the
coarse propagated beams into the exact waves you specified in `tilts.csv`.
