# Ray Data 2.50 multimodal inference benchmarks

The benchmark code behind
[Ray Data vs. Daft: Benchmarking multimodal AI workloads](https://www.anyscale.com/blog/ray-data-daft-benchmarking-multimodal-ai-workloads),
which compares [Ray Data](https://docs.ray.io/en/latest/data/data.html) against
[Daft](https://www.daft.ai/) 0.6.2 on five multimodal inference pipelines.

Published results live in the Ray docs under
[Ray Data benchmarks](https://docs.ray.io/en/latest/data/benchmark.html).

These scripts used to live in the Ray repo as release tests under
`release/nightly_tests/multimodal_inference_benchmarks`, added in
[ray-project/ray#57111](https://github.com/ray-project/ray/pull/57111). 

## Workloads

Each directory holds one pipeline, implemented twice — once with Ray Data and
once with Daft — so the two run the same work on the same cluster.

| Directory | Pipeline | Data |
| --- | --- | --- |
| [`image_classification`](image_classification) | Download images, decode, classify with ResNet-18 | 803,580 ImageNet images |
| [`large_image_embedding`](large_image_embedding) | Base64-decode and preprocess images, embed with ViT-base | ~4 TiB of images |
| [`document_embedding`](document_embedding) | Download PDFs, extract page text, chunk, embed with all-MiniLM-L6-v2 | 10k Digital Corpora PDFs |
| [`audio_transcription`](audio_transcription) | Resample audio, extract features, transcribe with Whisper-tiny | 113,800 Common Voice 17 files |
| [`video_object_detection`](video_object_detection) | Decode frames, detect objects with YOLO11n, crop detections | 1,000 Hollywood2 videos |

Every directory contains:

- `ray_data_main.py`ㅣ the Ray Data implementation.
- `daft_main.py` — the Daft implementation. Most are adapted from
  [Daft's own benchmarks](https://github.com/Eventual-Inc/Daft/tree/9da265d8f1e5d5814ae871bed3cee1b0757285f5/benchmarking/ai);
  the header comment of each file names the exact source.
- `requirements.txt` — pinned dependencies, including `ray[data]==2.50` and `daft==0.6.2`.
- `compute.yaml` — the [Anyscale compute config](https://docs.anyscale.com/configuration/compute-configuration/)
  describing the cluster the workload ran on.

`benchmark.py` at the repo root is the timing harness the Ray Data scripts
import, copied from Ray's `release/nightly_tests/dataset/benchmark.py`. It
records wall-clock runtime plus peak object store usage and spill, and writes
them as JSON to `$TEST_OUTPUT_JSON` (default `./result.json`). The Daft scripts
print their runtime to stdout instead.

## Results

From the [Ray docs benchmark page](https://docs.ray.io/en/latest/data/benchmark.html),
on the larger instance types:

| Workload | Ray Data 2.50 | Daft 0.6.2 |
| --- | --- | --- |
| Image Classification | 111.2 ± 1.2s | 195.3 ± 2.5s |
| Document Embedding | 29.4 ± 0.8s | 51.3 ± 1.3s |
| Audio Transcription | 312.6 ± 3.1s | 510.5 ± 10.4s |
| Video Object Detection | 623 ± 1.4s | 735.3 ± 7.6s |
| Large Image Embedding | 105.81 ± 0.79s | 752.75 ± 5.5s |

Instance size changes the picture, and that's the point the blog post makes. On
`g6.xlarge` (4 CPUs per GPU) Daft wins two of the workloads: image
classification, 315.0s to Ray Data's 456.2s, and video object detection, 758.8s
to 922s. Both reverse as CPU count rises: on `g6.8xlarge` (32 CPUs per GPU) Ray
Data leads 111.2s to 195.3s and 623s to 771.3s. Ray Data's advantage comes from
overlapping CPU preprocessing with GPU inference, so it grows with the
CPU-to-GPU ratio.

The blog post measured Ray Data 2.49.2 and a 2.50 release candidate; the docs
table reports 2.50.

## Reproducing

These workloads read multi-terabyte datasets on multi-node GPU clusters, so they
aren't meant to run on a laptop. All input data is publicly readable from S3.

**1. Start a cluster.** Use the workload's `compute.yaml`, the published numbers
depend on the instance types and node counts it pins. Note that
`large_image_embedding` asks for a much bigger cluster (40 GPU nodes plus 64 CPU
nodes) than the others (8 GPU nodes).

These files come from Ray's release test tooling, which substituted the cloud
name at runtime, so replace this line with your own Anyscale Cloud name before
using the file:

```yaml
cloud: {{env["ANYSCALE_CLOUD_NAME"]}}
```

**2. Install dependencies** on the cluster from the workload's requirements file:

```bash
pip install -r image_classification/requirements.txt
```

**3. Pick an output bucket.** Every script writes to
`s3://ray-data-write-benchmark/<random hex>`, which you almost certainly can't
write to. Edit `OUTPUT_PATH` (or `OUTPUT_PREFIX`) at the top of the script to
point at a bucket you own.

**4. Run a workload** from the repo root, so `benchmark.py` is importable:

```bash
PYTHONPATH=. python image_classification/ray_data_main.py
```

```bash
PYTHONPATH=. python image_classification/daft_main.py
```

The Ray Data run leaves its metrics in `result.json`; the Daft run prints
`Runtime: <seconds>`.

Comparing runs is only meaningful when both sides ran on the same cluster shape
with the same pinned versions. Newer Ray Data releases change the numbers. Some
scripts carry `NOTE` comments about behavior that improved in Ray Data 2.51 and
later.
