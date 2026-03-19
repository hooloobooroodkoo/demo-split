# New functionality for getting partial result

## def split_fileset(...)
The implementation can be found [here](https://github.com/hooloobooroodkoo/coffea/blob/split_strategy/src/coffea/util.py#L72-L91)

### Description of function

```python
def split_fileset(fileset, strategy=None, datasets=None, percentage=None):
    """
    Split the fileset into chunks to enable getting a partial result if one or several
    of the chunks failed to produce a result while being processed.
    One chunk is one partial fileset(unique combination of files), these are not usual coffea chunks.
    
    
    Input
    fileset:    {dataset: {"files": {path: treename, ...}}}
    strategy:   "by_dataset" — one dataset is one chunk; None — all datasets together
    percentage: integer that divides 100 evenly (20, 25, 50...).
                Each chunk gets this percentage of each dataset's files.
    datasets: list, callable or tuple of datasets' names
    
    Output
    List of fileset dicts
    If strategy only:
        chunks = _split_fileset(fileset, "by_dataset") - one chunk per dataset
    If percentage only:
        chunks = _split_fileset(fileset, percentage=50) - 2 chunks (50 of each dataset in 1st chunk and 2nd, mixed chunks
    If strategy and percentage:
        chunks = _split_fileset(fileset, "by_dataset", percentage=50) - N_datasets * 2 chunks, not mixed chunks
    If datasets + any/nothing:
        strategies are only applied to chosen datasets
    """
```
### How to run

If you want to run split_fileset function alone use the [branch `split_strategy`](https://github.com/hooloobooroodkoo/coffea/tree/split_strategy) \
with `example_split_simple.ipynb`:

```python
# concept example
chunks = split_fileset(fileset, strategy="by_dataset", percentage=20)
result = None
for chunk in chunks:
    try:
        output, metrics = run(chunk, processor_instance=Processor())
        if result is None:
            result = output
        else:
            result += output
    except BaseException as e:
        print(f"Error: {e}")
        continue
```
---

## def hash_fileset(...)
The implementation can be found [here](https://github.com/hooloobooroodkoo/coffea/blob/split_strategy/src/coffea/util.py#L53-L69), same branch `split_strategy`.

### Description of function

```python
def hash_fileset(chunk):
    """
    Return a stable SHA-256 hash for a fileset chunk.
    The hash considers dataset names, file paths in sorted order.

    Input
    chunk: fileset dict  {dataset: {"files": {path: treename, ...}, ...}, ...}

    Output
    hex string uniquely identifying this chunk's file content
    """
```
### How to run
If you want to run split_fileset and preserve partial result in Jupyter Notebook use this [branch](https://github.com/hooloobooroodkoo/coffea/tree/split_strategy) \
with `example_split_cache.ipynb`:

```python
# concept example
chunks = split_fileset(fileset, strategy="by_dataset", percentage=20)

import os

result = None
cache_dir = "./chunk_cache"
os.makedirs(cache_dir, exist_ok=True)
```
After the first run, partial result will be saved. Then user can just rerun the same cell and partial result will be extracted from cache, while processor run will be only applied to missing chunks.
```python
for chunk in chunks:
    chunk_hash = hash_fileset(chunk)
    cache_path = os.path.join(cache_dir, f"{chunk_hash}.coffea")

    if os.path.exists(cache_path):
        print(f"Loading cached result for chunk {chunk_hash}")
        output = load(cache_path)
    else:
        try:
            output, metrics = run(chunk, processor_instance=Processor())
            save(output, cache_path)
            print(f"Saved result for chunk {chunk_hash}")
        except BaseException as e:
            print(f"Error processing chunk {chunk_hash}: {e}")
            continue

    if result is None:
        result = output
    else:
        result += output
```
---

## class Result()
The implementation can be found [here](https://github.com/hooloobooroodkoo/coffea/blob/processor_result_type/src/coffea/util.py#L53C1-L63C8). It's a different branch called `processor_result_type`. \
proccessor.Runner always returns Result object (either Ok(Accumulatable) or Err(Exception)). Then analysis can be written in the following way - without \
catching and avoiding of Exception(s) but with the user being able to decide what to do with the chunk that failed to be processed.
```python
import os

result = None
cache_dir = "./chunk_cache"
os.makedirs(cache_dir, exist_ok=True)
```
```python
# percentage=20, 5 mixed chunks: 1st chunk is 20% of SingleMu_0 + 20% of SingleMu_1 ...
for chunk in chunks:
    chunk_hash = hash_fileset(chunk)
    cache_path = os.path.join(cache_dir, f"{chunk_hash}.coffea")

    if os.path.exists(cache_path):
        print(f"Loading cached result for chunk {chunk_hash}")
        output = load(cache_path)
    else:
        run_result = run(chunk, processor_instance=Processor())
        if run_result.is_ok():
            output, metrics = run_result.unwrap()
            save(output, cache_path)
            print(f"Saved result for chunk {chunk_hash}")
        else:
            # user can implement their own logic on how to treat failed chunks
            print(f"Error processing chunk {chunk_hash}: {run_result.exception}")
            continue

    if result is None:
        result = output
    else:
        result += output
```

### Description of Result, Ok, Err

- Processor run returns an object of calss Return — either Ok(Accumulatable) or Err(Exception).
- result.is_ok() - check success whether Result is Ok or Err
- result.unwrap() - to get the value (Accumulatable or Exception)
- result.exception - to inspect the error if Result is Err
- When savemetrics=True, the wrapped value is (output, metrics)
Implementation can be found [here](https://github.com/hooloobooroodkoo/coffea/blob/processor_result_type/src/coffea/processor/result.py) and [here](https://github.com/hooloobooroodkoo/coffea/blob/processor_result_type/src/coffea/processor/executor.py#L1580)
This implementation exist on a different branch. Therefore, to run `example_split_result_type.ipynb`, [this branch](https://github.com/hooloobooroodkoo/coffea/tree/processor_result_type) should be used.





