I built a microscopy counting tool because a total count alone did not tell me which nuclei a method had missed or joined together. I wanted to compare two counting methods and inspect their mistakes on the original images.

The tool marks individual nuclei, compares them with public human annotations, and flags images or objects that need a closer look. It counts nuclei as a proxy for cells; it does not measure whole cells.

**Try it without installing anything:** [browse the saved image-by-image report](reports/bbbc039/gallery.md). Each image has a reference panel, both methods' error overlays, counts, and review flags. For a filterable offline version, [download index.html](https://github.com/jackaI1/microscopy-count-qc/raw/refs/heads/main/reports/bbbc039/index.html) and open the saved file in a browser. All its images are embedded.

Green outlines mark matched predictions, pink marks extra or poorly overlapping predictions, and orange marks missed reference nuclei. Blue boxes mark possible merged objects for human review. These warnings are suggestions to inspect an image, not confirmed biological problems.

## What I found

I ran both methods on all 50 images in the publisher's test split, containing 5,720 annotated nuclei. The table comes from `uv run microscopy-qc audit`, which recomputes the scores from the [saved label arrays](reports/bbbc039/labels) and checks them against [results.json](reports/bbbc039/results.json).

| Method | F1 at overlap 0.50 | F1 at overlap 0.75 | F1 at overlap 0.90 | Mean absolute count error per image |
|---|---:|---:|---:|---:|
| Threshold + watershed | 0.8846 | 0.8040 | 0.5467 | 7.54 nuclei |
| Pretrained Cellpose nuclei | 0.9446 | 0.9269 | 0.7848 | 10.30 nuclei |

F1 balances missed and extra objects, with a perfect score of one. Overlap means intersection over union, or IoU: the shared area divided by the combined area. A higher overlap cutoff demands a closer outline. These are micro F1 scores, calculated by pooling matched, missed, and extra objects across images.

Cellpose produced closer outlines overall, but watershed had the smaller average count error. That matters because a plausible total can hide missed objects and extra detections that cancel each other out. I keep both measures and the individual images visible rather than choosing a winner from one number.

Both methods triggered focus review on 6 test images. Possible merges were flagged in 24 watershed images and 29 Cellpose images. Those are heuristic flags, not measured focus or merge accuracy: the dataset has no corresponding review labels. The [per-image CSV](reports/bbbc039/per_image.csv) includes signed and absolute count errors, F1 at each cutoff, focus scores, and flagged object IDs.

## Run it myself

With [uv](https://docs.astral.sh/uv/) installed, from this repository:

```sh
uv sync --frozen --extra model --extra dev
uv run --frozen --extra model microscopy-qc download
uv run --frozen --extra model microscopy-qc run
uv run --frozen microscopy-qc audit
```

The download command retrieves the original public images, annotations, and split lists. It checks their SHA256 checksums, which identify the exact file contents. The run command also checks extracted files against those archives before claiming an official dataset run. Data and downloaded model weights stay out of git. The pipeline uses CPU inference with two threads by default.

The saved report goes to `reports/bbbc039/`. To make a separate quick trial:

```sh
uv run --frozen --extra model microscopy-qc run --limit 2 --output reports/trial
```

That output records that it is a partial run. To use only the classical method, pass `--methods watershed`; it does not need the model dependencies. To rebuild report formatting from saved predictions without rerunning inference:

```sh
uv run --frozen microscopy-qc render
```

## How I kept the comparison fair

I use the publisher's whole-image partitions: 100 training images, 50 validation images, and 50 test images. No random patches cross between partitions. I use only training annotations to estimate nucleus diameter, a minimum object area, and review cutoffs. Validation images are left unused. Neither method is fitted to test annotations, and the pretrained model is not fine-tuned.

The baseline smooths the image, chooses a foreground threshold, and uses watershed, which separates touching objects around peaks in their distance from the background. Cellpose uses its frozen pretrained nuclei model. Both receive the same training-derived diameter and minimum object area. The exact parameters, dependency versions, input hashes, model-weight hash, and complete split membership are saved with the results.

For each overlap cutoff, I match predicted and annotated nuclei one to one, maximizing the number of valid matches before breaking ties by overlap. An unmatched prediction is an extra detection; an unmatched annotation is a miss. The report also saves macro F1, which averages each image's F1, so sparse and crowded images can be compared.

Focus review uses an edge-sharpness score below the training distribution's fifth percentile. Possible merges have unusually large area or multiple separated interior peaks. These rules use the image and predicted objects, not test annotations. A separate synthetic blur check in the results records whether deliberate smoothing reduces the focus score; it is excluded from the benchmark scores.

## Limits I would keep in mind

This is a demonstration on one public fluorescence dataset, not a validated laboratory counting system. Treatments can change nucleus size and shape and trigger false review flags. Dividing, overlapping, or fragmented nuclei complicate the link between nucleus count and cell count. Review warnings should guide a person, not automatically discard a sample.

The whole-image holdout protects my own calibration, but I cannot establish independence from Cellpose's original pretraining. BBBC039 partly overlaps BBBC038, another public collection. I report the pretrained model's performance on these images, not a claim that it never saw related images. Results may vary slightly across hardware or dependency changes; `uv.lock` and saved provenance make those differences traceable.

## Public sources

I used the original sources below, with no substitute dataset and no private data.

- [BBBC039v1](https://bbbc.broadinstitute.org/BBBC039), Caicedo et al. 2018, from the Broad Bioimage Benchmark Collection. The dataset page describes the images, annotations, official partitions, and CC0 release. Its linked [study](https://doi.org/10.1101/335216) describes the annotation and segmentation evaluation context.
- [Broad Bioimage Benchmark Collection citation](https://doi.org/10.1038/nmeth.2083): Ljosa, Sokolnicki and Carpenter, *Nature Methods*, 2012.
- Original [images](https://data.broadinstitute.org/bbbc/BBBC039/images.zip), [masks](https://data.broadinstitute.org/bbbc/BBBC039/masks.zip), and [metadata](https://data.broadinstitute.org/bbbc/BBBC039/metadata.zip). Download URLs and pinned archive hashes are in [sources.json](sources.json).
- [Publisher's annotation decoder](https://gist.github.com/jccaicedo/15e811722fca51e3ae90e8b43057f075). I follow its first-channel connected-region convention rather than treating the RGB annotation as ordinary instance IDs.
- [Cellpose](https://github.com/MouseLand/cellpose), using version 3.1.1.1 and its pretrained nuclei weights. The [model documentation](https://cellpose.readthedocs.io/en/v3.1.1.1/models.html) explains the model and public weight downloads; the [Cellpose paper](https://doi.org/10.1038/s41592-020-01018-x) describes the method. Cellpose code is BSD-3-Clause; see its upstream license and model terms.

## Code and checks

[Data loading](src/microscopy_qc/data.py), [segmentation](src/microscopy_qc/segment.py), [matching and metrics](src/microscopy_qc/metrics.py), [review rules](src/microscopy_qc/qc.py), and [report generation](src/microscopy_qc/report.py) are separate modules. [Design notes](docs/design.md) explain the evaluation choices.

```sh
uv run --frozen --extra dev pytest -q
uv run --frozen --extra dev ruff check src tests
uv run --frozen --extra dev ruff format --check src tests
uv run --frozen microscopy-qc audit
```

The test suite has 25 tests covering matching, blank images, touching nuclei, blur sensitivity, merge warnings, unsafe archives, changed extracted data, report output, and the command-line workflow. [CI](.github/workflows/ci.yml) also audits all 100 saved image/method rows and downloads the public data to run both real methods on one held-out image.
