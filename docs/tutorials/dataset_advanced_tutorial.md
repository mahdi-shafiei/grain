--------------------------------------------------------------------------------

jupytext: formats: ipynb,md:myst text_representation: extension: .md
format_name: myst format_version: 0.13 jupytext_version: 1.16.6 kernelspec:
display_name: Python 3

## name: python3

+++ {"id": "9ufbgPooUPJr"}

# Advanced `Dataset` usage

If you decided to use `Dataset` APIs, there's a good chance you want to do one
or more processing steps described in this section, especially if working on
data ingestion for generative model training.

``` {code-cell}
:id: OFw1tjvkP3wb

# @test {"output": "ignore"}
!pip install grain
# @test {"output": "ignore"}
!pip install tensorflow_datasets
```

``` {code-cell}
:id: 8uUXhV-njM7U

import grain.python as grain
import tensorflow_datasets as tfds
```

+++ {"id": "Qm0Ob5ATHhhL"}

## Checkpointing

We provide `GrainCheckpoint{Save|Restore}` to checkpoint the
`DatasetIterator`. It is recommended to use it with
[Orbax](https://orbax.readthedocs.io/en/latest/index.html), which can checkpoint
both, input pipeline and model, and handles the edge cases for distributed
training.

``` {code-cell}
:id: dvVPaXTUHhhL

ds = (
    grain.MapDataset.source(tfds.data_source("mnist", split="train"))
    .seed(seed=45)
    .shuffle()
    .to_iter_dataset()
)

num_steps = 4
ds_iter = iter(ds)

# Read some elements.
for i in range(num_steps):
  x = next(ds_iter)
  print(i, x["label"])
```

``` {code-cell}
:id: lg9BViluLQ3d

!pip install orbax
```

``` {code-cell}
:id: SUX9L1IMHhhL

import orbax.checkpoint as ocp

mngr = ocp.CheckpointManager("/tmp/orbax")

!rm -rf /tmp/orbax

# Save the checkpoint.
assert mngr.save(
    step=num_steps, args=grain.GrainCheckpointSave(ds_iter), force=True
)
# Checkpoint saving in Orbax is asynchronous by default, so we'll wait until
# finished before examining checkpoint.
mngr.wait_until_finished()

!ls -R /tmp/orbax
```

``` {code-cell}
:id: c8DJCPZgLQ3d

!cat /tmp/orbax/*/*/*.json
```

``` {code-cell}
:id: Zb5TrmjrHhhL

# Read more elements and advance the iterator.
for i in range(4, 8):
  x = next(ds_iter)
  print(i, x["label"])
```

``` {code-cell}
:id: 1PA1rpnwHhhL

# Restore iterator from the previously saved checkpoint.
mngr.restore(num_steps, args=grain.GrainCheckpointRestore(ds_iter))
# Iterator should be set back to start from 4.
for i in range(4, 8):
  x = next(ds_iter)
  print(i, x["label"])
```

+++ {"id": "GfA_bctscNyV"}

## Mixing datasets

`Dataset` allows mixing multiple data sources with potentially different
transformations. There's two different ways of mixing `Dataset`s:
`MapDataset.mix` and `IterDataset.mix`. If the mixed `Datasets` are sparse (e.g.
one of the mixture components needs to be filtered) use `IterDataset.mix`,
otherwise use `MapDataset.mix`.

``` {code-cell}
:id: fwvOt8-cqcQn

import pprint
import numpy as np
```

``` {code-cell}
:id: e8ROZXhtwOx3

tfds.core.DatasetInfo.file_format = (
    tfds.core.file_adapters.FileFormat.ARRAY_RECORD
)
# This particular dataset mixes medical images with hand written numbers,
# probably not useful but allows to illustrate the API on small datasets.
source1 = tfds.data_source(name="pneumonia_mnist", split="train")
source2 = tfds.data_source(name="mnist", split="train")
ds1 = grain.MapDataset.source(source1).map(lambda features: features["image"])
ds2 = grain.MapDataset.source(source2).map(lambda features: features["image"])
ds = grain.MapDataset.mix([ds1, ds2], weights=[0.7, 0.3])
print(f"Mixed dataset length = {len(ds)}")
pprint.pprint(np.shape(ds[0]))
```

+++ {"id": "crR2FZ1Gf6-O"}

If filtering inputs to the mixture, use `IterDataset.mix`.

``` {code-cell}
:id: DTmUbvK4r8T8

source1 = tfds.data_source(name="pneumonia_mnist", split="train")
source2 = tfds.data_source(name="mnist", split="train")
ds1 = (
    grain.MapDataset.source(source1)
    .filter(lambda features: int(features["label"]) == 1)
    .to_iter_dataset()
)
ds2 = (
    grain.MapDataset.source(source2)
    .filter(lambda features: int(features["label"]) > 4)
    .to_iter_dataset()
)

ds = grain.IterDataset.mix([ds1, ds2], weights=[0.7, 0.3]).map(
    lambda features: features["image"]
)
pprint.pprint(np.shape(next(iter(ds))))
```