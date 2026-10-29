# Coding for AWARE: filesystem, concurrency, and Slurm guidance

Use this context when designing, reviewing, or modifying code that will run on the AWARE HPC cluster at CW3E / Scripps Oceanography / UC San Diego.

Optimize the complete workflow under realistic concurrent use. A script that is fast by itself may perform poorly when many copies run at once.

## Site context

- AWARE uses Slurm. Many small jobs, array tasks, or copies of the same program may run concurrently, alongside other users' workloads.
- Login through `login.aware.sdsc.edu`. Login nodes have strict limits; run computation, builds, substantial analysis, and interactive development inside Slurm jobs on compute nodes.
- Shared project storage is BeeGFS, mounted at `/cw3e/mead`, with projects under `/cw3e/mead/projects`. Keep project-specific input and output paths configurable.
- Home directories have a 20 GB quota. Keep large environments, datasets, temporary data, and outputs in the project area. Mead has no quotas or backups; usage is monitored and project cleanups may be requested. Copying an output to Mead does not create a backup.
- The `awr-2-*` nodes in `shared-128` and `compute-128` have node-local storage at `/scratch/${SLURM_JOB_USER}`, with approximately 295 GiB total filesystem capacity; an example node showed 280 GiB available. This capacity is shared by jobs on that node. Available space varies; do not treat it as a private per-job allocation or assume the same local storage on other node types.
- The filesystem has six backend storage servers. Data is stored on HDDs in large storage arrays using ZFS, with server-side caching.
- BeeGFS uses RAID0 striping across 4 storage targets with 512 KiB stripe chunks. These are fixed site defaults that users cannot change; assume them when coding for AWARE without requesting routine verification. Do not interpret stripe width as a limit of four workers or four simultaneous jobs.
- Metadata servers can keep up with normal directory scans. Do not assume metadata is the primary bottleneck or prohibit ordinary file discovery. Avoid having every worker repeat the same unnecessary discovery.
- Each client has a 100 Gb/s HDR InfiniBand connection. Each storage server has two 200 Gb/s InfiniBand connections, but effectively one is used. Do not budget 400 Gb/s per server or equate network link rates with sustained HDD throughput.
- BeeGFS clients use `buffered` caching: small buffers for read-ahead and write-back. Do not assume this provides a large Linux page cache holding previously read files in client RAM.
- Server caches can help with reuse, but their capacity is shared and finite. Prefer finishing the useful work on a file or bounded file batch while its data is being accessed, then moving on. Returning to files after a large intervening working set can lose that cache benefit.

## Choose Slurm resources for AWARE

### Seasonal availability

From roughly September through April, the 192-core nodes are reserved for near-real-time (NRT) operations. For general work during that period, expect to use `shared-128` or `compute-128`. Do not select `shared-192` or `compute-192` as a general-work fallback during the NRT season.

A job may use the reserved nodes only when it is authorized to run inside the NRT reservation. Include `--reservation=<name>` only when the user has supplied the appropriate reservation and confirmed that the job belongs there; do not invent a reservation name. Outside the season, use 192-core nodes only when they suit the workload and are available.

The wiki also notes that about half the 128-core pool is reserved during NRT season. Plan for reduced general capacity and realistic queue waits; do not promise immediate starts.

### Partition and resource requests

| Partition | Allocation behavior | Typical general-work use |
| --- | --- | --- |
| `shared-128` | Allocates part of a 128-core node; other jobs can share it | Serial programs, modest worker pools, threaded analysis, and interactive work |
| `compute-128` | Reserves entire 128-core nodes even when fewer cores are requested | Work needing whole nodes, efficient full-node batches, or multi-node MPI |

- Choose `shared-128` for work needing only part of a node. Moving a small job to `compute-128` reserves a full node without automatically making the program use it.
- Use the user's actual project ID with `--account` / `-A`. AWARE project IDs are the same as on Expanse. Use the applicable site QOS; `shared-128` is the QOS used with the wiki's shared-128 development jobs. Do not invent account or QOS names.
- The maximum wall time is 7 days per job. Request a realistic wall time for the work; the maximum is not the default recommendation.
- Slurm defaults to approximately 2 GB of memory per requested core. Memory requests also affect billing: the wiki's example of one CPU with `--mem=16G` is billed as eight cores. Size CPU and memory requests together.
- Distinguish processes from threads: `--ntasks` describes tasks such as MPI ranks; `--cpus-per-task` allocates CPUs to each task. A single Python program with a local worker pool normally uses one task with an appropriate CPU count. Requesting more tasks alone does not parallelize it.
- `overcommit-128` and `overcommit-192` are for operational workflows. Do not use them for general job scripts.
- Use one node type and partition for an MPI job; do not span the 128-core and 192-core pools.
- AWARE allocation accounting uses CPU core minutes. Use `aware_client -A <PROJECT>` to inspect the intended project; do not interpret its allocation values as core hours.

Example resource layout for a single-node program using four CPUs and 8 GB of memory; replace the account and program, activate the required environment, and adjust resources to the actual workload:

```bash
#!/bin/bash
#SBATCH --job-name=analysis
#SBATCH --account=<PROJECT>
#SBATCH --partition=shared-128
#SBATCH --qos=shared-128
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --time=01:00:00

srun python workflow.py
```

The program must actually use the allocated CPUs, and its worker/thread counts must follow the concurrency guidance below. A serial program should request the resources it uses.

## Environments, MPI, and interactive development

- Load modules in order: compiler, MPI when needed, then dependent libraries. Use `module spider <name>` to discover dependencies. Legacy `cpu/...` modules are not part of the normal user workflow.
- The wiki recommends `intel/2023.2.4.31` and `intel-mpi` for its Intel examples. Preserve the application's working compiler/MPI stack and load only what it needs; do not assume every Python job needs MPI.
- MPI/internode communication uses `mlx5_1`; BeeGFS storage uses `mlx5_0`. For UCX-based MPI, the wiki specifies `export UCX_NET_DEVICES=mlx5_1:1`. Apply this to the appropriate MPI workflow.
- Conda environment startup on BeeGFS may be slow on first use and faster with caching. Consider packaging an environment into a container for repeated startup and portability when appropriate. Lustre `lfs` tuning tricks do not apply to BeeGFS.
- Run Jupyter and VS Code services inside Slurm allocations. For VS Code, use the maintained launcher `/cw3e/mead/datasets/cw3e/sbin/code-tunnel/launch_code.sh` and Remote Tunnels. Keep one tunnel job per user; cancel unused development jobs when finished.

## Design the file access before choosing parallelism

Briefly explain:

1. The unit of work: file, group of files, time window, variable, location, or another unit.
2. Which input files each unit reads, how much it reads, and which units overlap.
3. How many jobs, processes, threads, and simultaneous file readers/writers the proposed workflow creates.
4. Where data is reused, the approximate peak memory per worker and per node, and who owns each output.

Ask a focused question if missing information would materially change the design. Otherwise state reasonable assumptions and proceed. Do not invent site paths, scratch capacity, scheduler limits, or measured throughput.

## Prefer locality and useful reuse

- Prefer substantial sequential or contiguous reads and writes where the data layout permits. HDD-backed storage can suffer when many workers interleave small, scattered requests.
- Group related calculations that use the same input. Read the necessary data once, compute the relevant products, and release it when finished.
- Keep needed data in application memory when it fits within the Slurm memory allocation. If it does not fit, process bounded chunks or file batches. Do not load an entire large file just to obtain a small subset.
- Keep files open for the useful work within a processing unit, then close them promptly. Avoid repeated open/read/close cycles for the same data and avoid holding the entire dataset open unnecessarily.
- Match access to the actual file format and storage layout. For NetCDF/HDF5/xarray/Dask workflows, consider dimensions, compression, on-disk chunks, application chunks, and task count together. BeeGFS stripe width does not determine the appropriate application chunk shape.
- Account for algorithmic needs. Calculations spanning files may need adjacent files, overlap, or a streaming reduction; preserve scientific results when reorganizing the work.

Example: if 100 requested locations all need data from the same collection of model files, consider assigning a file or bounded file batch to a worker and extracting all 100 locations together. A separate job for every location may repeatedly read the same files. Compare the actual bytes read and memory use before choosing.

## Control concurrency across the workflow

- More workers are useful only while they improve aggregate useful throughput. Choose I/O concurrency separately from the number of available CPU cores.
- For many short tasks with overlapping inputs, consider batching them into one appropriately sized Slurm job with a bounded worker pool, or a smaller number of jobs processing coherent batches.
- Bundling alone does not improve I/O: launching the same number of competing readers inside one allocation can preserve the original problem. The benefit must come from less duplicate reading, better locality, controlled concurrency, or reduced task startup cost.
- Keep independent jobs or arrays when they suit the workload, particularly for substantial computation on disjoint inputs. Make concurrency configurable.
- For arrays, use an explicit simultaneous-task limit, such as `--array=0-99%4`. Here 4 is only an example, not a recommended AWARE limit. An array limit applies to that array; several arrays can multiply total concurrency.
- Count nested parallelism: concurrent jobs × processes per job × threads per process. Set library thread counts deliberately and keep CPU and memory use within the allocation.
- Multiple nodes require explicit work distribution. A local process pool runs only on its launching node; requesting more nodes does not distribute it automatically.

## Outputs and temporary data

- Give workers distinct output and temporary paths. Use one writer per ordinary output file unless the selected format, library, and implementation explicitly support coordinated parallel writes.
- Batch tiny output records where practical. Choose output grouping that also supports the next analysis step, rather than creating avoidable rereads or an oversized shared output bottleneck.
- On `awr-2-*` nodes, consider `/scratch` for reused inputs and repeated intermediate I/O. Check available space and budget for staged inputs, temporary data, and outputs across concurrent workers. Use job/task-specific subdirectories to avoid collisions with other jobs.
- Each node has its own `/scratch`; the same pathname on another node does not refer to the same data. Coordinate staging so workers within a job reuse a local input copy where practical. Include staging costs; copying the same large dataset separately for every small task can increase total BeeGFS traffic.
- Treat local scratch as temporary: copy retained results to BeeGFS before the job exits and clean up only the temporary directories created by that job. Do not assume an automatic cleanup schedule. A directory called `scratch` under a BeeGFS project is still shared storage; `/tmp` and `$TMPDIR` are not established substitutes for `/scratch`.

## Verify at the intended scale

Use a small representative workload to compare a few concurrency levels. Report end-to-end elapsed time, useful work completed per unit time, and peak memory. Include input discovery, reads, computation, writes, and any staging.

Evaluate performance with realistic competing readers and writers when practical. A fast repeated run may be benefiting from server cache; do not assume it represents uncached or sustained performance. Never clear system caches for a benchmark.

Recommend concurrency from these measurements and the workload's data reuse, rather than from stripe width, CPU count, or link speeds alone.

## Public references

The AWARE hardware and site configuration above were supplied by the site administrator. Scheduling, environment, and storage guidance also comes from the supplied CW3E AWARE wiki PDF. The administrator's current instructions take precedence over older wiki text, including fixed striping defaults and seasonal partition availability. These references explain the general BeeGFS and Slurm behavior:

- [BeeGFS client caching modes](https://doc.beegfs.io/latest/advanced_topics/client_caching.html)
- [BeeGFS striping](https://doc.beegfs.io/8.3/advanced_topics/striping.html)
- [Slurm job arrays and simultaneous-task limits](https://slurm.schedmd.com/job_array.html)

