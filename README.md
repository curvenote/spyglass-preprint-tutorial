# spyglass-preprint-tutorial

A runnable environment for the Spyglass tutorials that accompany the preprint:

> Lee, K.H.\*, Denovellis, E.L.\*, … Frank, L.M. (2024). *Spyglass: a data analysis framework for reproducible and shareable neuroscience research.* bioRxiv. [10.1101/2024.01.25.577295](https://doi.org/10.1101/2024.01.25.577295)

Launch it on the Frank Lab BinderHub:

<https://hhmi-frank-4eg38zi9.curvenote.dev/v2/gh/curvenote/spyglass-preprint-tutorial/HEAD>

Then open `notebooks/` and work through the notebooks **in order**. Start with `00_Spyglass_Smoke_Test.ipynb`, which checks that the server is ready. Later notebooks build on what earlier ones put in the database.

## Spyglass version

The image installs Spyglass at commit [`6122f07`](https://github.com/LorenFrankLab/spyglass/commit/6122f076918cb0bbda220d71419848e9d71150a7) (22 Jan 2026, version `0.5.5a2.dev33`). That's the commit [curvenote/spyglass](https://github.com/curvenote/spyglass) tags as `snapshot/biorxiv-v5`: the code described in bioRxiv v5 of the preprint. The notebooks are copied from the same commit.

## Notebooks

The upstream tutorials use two example files: `minirec20230622.nwb` (240 MB, a 42-second recording) and `mediumnwb20230802.nwb` (32 GB). The 32 GB file doesn't fit on this hub's 16 GB shared disk, so every notebook here uses **minirec**.

| Notebook | Status on this hub |
| --- | --- |
| `00_Spyglass_Smoke_Test` | Hub-specific: checks packages, database, Spyglass imports and the NWB file. Replaces upstream `00_Setup`, which is about installing Spyglass locally. |
| `01_Concepts` | Runs as upstream. |
| `02_Insert_Data` | Runs. The final "real data" example needs a 5 GB DANDI file that isn't on the shared drive, so it is skipped with a message. |
| `04_Merge_Tables` | Runs. The last cell deletes the session after asking you to confirm. **Answer no** if you want to continue with the later notebooks. |
| `05_Export` | Runs. The DANDI upload step is skipped unless you add your own DANDI account details. |
| `10_Spike_SortingV1` | Runs. Making the FigURL curation view needs a kachery account, so it's off by default. |
| `20_Position_Trodes`, `24_Linearization`, `30_LFP`, `31_Theta` | Run. |
| `40_Extracting_Clusterless_Waveform_Features`, `41_Decoding_Clusterless`, `42_Decoding_SortedSpikes` | **Ported to minirec.** Every step runs, but with ~10 s of position data the decoding results are only illustrative. 40 uses 8 of the 32 tetrodes to keep its run time reasonable on 1 CPU. |

Every change from the upstream notebooks is marked with a `# Binder hub:` comment, or a **Binder hub:** note in the text.

**Not included yet:**
- `11_Spike_Sorting_Analysis`, `32_Ripple_Detection` and `50_MUA_Detection` need real behaviour and labelled CA1 tetrodes, which minirec doesn't have. They are waiting for a cut-down version of `mediumnwb20230802.nwb`.
- `03_Data_Sync` needs Frank Lab infrastructure.
- `10_Spike_SortingV0` is the legacy pipeline.
- `12_Burst_Merge_Curation`, `21`/`22` (DeepLabCut) and `60_MoSeq` need other datasets or a GPU.

## What is in the image

All build configuration is in [`binder/`](binder/). repo2docker reads only that folder.

| File | Purpose |
| --- | --- |
| `binder/environment.yml` | Python 3.9 + Spyglass at the commit above. Versions are pinned; changes from upstream's environment are marked `hub:`. |
| `binder/apt.txt` | OpenCV runtime libraries, graphviz, and the MySQL 8.0 server and client. |
| `binder/postBuild` | At build time: creates an empty MySQL data directory (user `root`, password `tutorial`) and links `~/.datajoint_config.json` to `dj_local_conf.json`. |
| `binder/start` | Every time a server starts: starts MySQL on `127.0.0.1:3306` and links NWB files from the shared data folder into `~/spyglass_data/raw`. |
| `binder/resolved-packages.txt` | For reference only: every conda, pip and apt package version in the tested image. repo2docker does not read it. |
| `dj_local_conf.json` | The DataJoint/Spyglass config the notebooks load (database login, data folders under `~/spyglass_data`). |

### Database

Spyglass needs a MySQL server. On the 2i2c hub, MySQL runs as a separate container next to each user. This hub can't do that, so a small MySQL 8.0 server runs **inside each user's container**:

- It only listens on `127.0.0.1`, so other users and pods can't reach it. The fixed `tutorial` password protects nothing beyond that container.
- It is tuned for the 2G memory limit (`performance_schema` off, 64 MB InnoDB buffer pool).
- **Nothing persists.** The database and everything under `~/spyglass_data` are deleted when the server stops (after 1 hour idle). Each new server starts with an empty database, so start again from `02_Insert_Data`.

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

## Building and testing locally

```bash
pip install jupyter-repo2docker==2026.4.0
repo2docker --no-run --image-name spyglass-binder .

# run one notebook headless with the hub's limits
docker run --rm --memory=2g --cpus=1 \
  -e SHARED_DATA_DIR=/home/jovyan/data -v /path/to/shared:/home/jovyan/data:ro \
  spyglass-binder \
  bash -c 'cd notebooks && jupyter nbconvert --to notebook --execute --output /tmp/out.ipynb 00_Spyglass_Smoke_Test.ipynb'
```
