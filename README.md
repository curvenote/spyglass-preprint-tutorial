# spyglass-preprint-tutorial

A runnable environment for the Spyglass tutorial that accompanies the preprint:

> Lee, K.H.\*, Denovellis, E.L.\*, … Frank, L.M. (2024). *Spyglass: a data analysis framework for reproducible and shareable neuroscience research.* bioRxiv. [10.1101/2024.01.25.577295](https://doi.org/10.1101/2024.01.25.577295)

Launch it on the Frank Lab BinderHub:

<https://hhmi-frank-4eg38zi9.curvenote.dev/v2/gh/curvenote/spyglass-preprint-tutorial/HEAD>

Then open **`00_Spyglass_Smoke_Test.ipynb`** and run all cells. It checks that the packages are installed, that the database is up, that the main Spyglass modules import, and that the tutorial NWB file is available.

## What is in the image

All build configuration is in [`binder/`](binder/). repo2docker reads only that folder.

| File | Purpose |
| --- | --- |
| `binder/environment.yml` | Python 3.9 + Spyglass 0.5.1 environment. Adapted from the 2i2c hub image [LorenFrankLab/hhmi-spyglass-image](https://github.com/LorenFrankLab/hhmi-spyglass-image); the changes are commented in the file. |
| `binder/apt.txt` | OpenCV runtime libraries, graphviz, and the MySQL 8.0 server and client. |
| `binder/postBuild` | At build time: creates an empty MySQL data directory (user `root`, password `tutorial`) and writes `~/.datajoint_config.json`. |
| `binder/start` | Every time a server starts: starts MySQL on `127.0.0.1:3306` and links the NWB files from the shared data folder into `~/spyglass_data/raw`. |
| `binder/resolved-packages.txt` | For reference only: every conda, pip and apt package version in the tested image. repo2docker does not read it. |

Direct dependencies in `environment.yml` are pinned to exact versions. Transitive conda/pip dependencies and the Ubuntu packages are not, so compare a new build against `resolved-packages.txt` to see what changed.

### Resource use (measured with a 2 GB / 1 CPU limit)

| | |
| --- | --- |
| Image | 7.0 GB on disk (1.7 GB compressed); the conda env is 3.6 GB |
| Build | ~4 min for the environment layer on a 4-CPU machine; expect roughly 6–10 min cold on the hub's 2-vCPU VM |
| Server start | ~4 s until Jupyter is up (MySQL starts in ~1 s) |
| Memory, idle | ~380 MiB (mysqld ~180 MiB, Jupyter ~145 MiB, rest page cache) |
| Memory, during the smoke test | ~1.0 GiB peak, including page cache |

### Database

Spyglass needs a MySQL server. The 2i2c hub runs one as a sidecar container next to each user. This hub can't do that, so a small MySQL 8.0 server runs **inside each user's container**:

- It only listens on `127.0.0.1`, so other users and pods can't reach it. The fixed `tutorial` password protects nothing beyond that container.
- It is tuned for the 2G memory limit (`performance_schema` off, 64 MB InnoDB buffer pool).
- **Nothing persists.** The database and everything under `~/spyglass_data` are deleted when the server stops (after 1 hour idle).

### Data

Notebooks read the tutorial data from the read-only shared folder (`$SHARED_DATA_DIR`, `/home/jovyan/data`). Admins should copy this file there:

| Shared path | Size | Source |
| --- | --- | --- |
| `spyglass/raw/minirec20230622.nwb` | 240,712,080 bytes (~230 MiB) | <https://ucsf.box.com/shared/static/k3sgql6z475oia848q1rgms4zdh4rkjn.nwb> (sha256 `637df8ea3c9d7c3b0c6c562bb2bcfd6ae7e8f99df979404a72c601da3bbfb09f`) |

```bash
mkdir -p <shared-disk>/spyglass/raw
curl -L -o <shared-disk>/spyglass/raw/minirec20230622.nwb \
  https://ucsf.box.com/shared/static/k3sgql6z475oia848q1rgms4zdh4rkjn.nwb
```

`start` links every `*.nwb` in `$SHARED_DATA_DIR/spyglass/raw/` (or directly in `$SHARED_DATA_DIR`) into `~/spyglass_data/raw/`. That folder is writable, so Spyglass can create its `<name>_.nwb` copy next to the original.

## Building locally

```bash
pip install jupyter-repo2docker==2026.4.0
repo2docker -v /path/to/shared:/home/jovyan/data .
```

To run the smoke test with the hub's limits and without a browser:

```bash
repo2docker --no-run --image-name spyglass-binder .
docker run --rm --memory=2g --cpus=1 \
  -e SHARED_DATA_DIR=/home/jovyan/data -v /path/to/shared:/home/jovyan/data:ro \
  spyglass-binder \
  jupyter nbconvert --to notebook --execute --output /tmp/out.ipynb 00_Spyglass_Smoke_Test.ipynb
```
