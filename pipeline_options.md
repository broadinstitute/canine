# Canine Pipelines

Canine's `Orchestrator` class is capable of reading a yaml file which lays out instructions
for how to configure and run one or more jobs on a slurm cluster. These pipeline
options may also be provided on the command line, so proper command line syntax
for each option will also be provided in each section

## name

The `name` key (`--name pipeline_name`) is optional, but provides a name for the
pipeline. Currently this is not used for anything other than naming the job on the
SLURM server.

## script

The `script` key (`--script path/to/script`) provides the path to a bash file to
be run. When provided in a yaml file, the `script` key may also be an array with
each line representing one bash command. For example:

```yaml
script:
  - sudo docker run --rm -v $CANINE_ROOT:$CANINE_ROOT my_docker_image my_docker_command
  - tar czf output.tar.gz *.outputs*
```

**Note:** This is a regular bash script, which means that if one command fails,
bash will continue to the next line. If you wish for the script to fail as soon
as one command fails, add `set -eo pipefail` as your first command

### Docker notes
* When using the `TransientGCP` backend, you **must** use `sudo` to use docker
* When using the `TransientGCP` backend with GPUS, you will need to add the option
`--runtime nvidia` option to enable access to the GPUS
* We recommend setting `-v $CANINE_ROOT:$CANINE_ROOT` as this will mount the pipeline
directory into the container. This allows for existing input variable filepaths to
function correctly in the container
* We recommend setting `--cpus $SLURM_CPUS_PER_TASK` and `--memory $(expr $SLURM_CPUS_PER_TASK '*' $SLURM_MEM_PER_CPU)MB`.
While these settings are entirely optional, they enforce CPU and memory usage limits
in the container, which can help avoid accidentally exceeding the resources provisioned
for each job

## inputs

The `inputs` key (`--inputs inputName:inputvalue`) specifies pipeline inputs.
When provided on the command line, the input name must also be provided (as shown
  above). You may provide the `--inputs` flag as many times as you wish (one argument
  each time), and may provide as many inputs to a single _inputName_ as necessary.
Exact handling of inputs is determined by the adapter (see below), but in general
an input name with one value will be held constant across all jobs in the pipeline,
whereas an input name with multiple values will take one of the given values in each job.
Inputs are passed to the adapter which determines how many jobs need to be launched,
and the exact inputs for each job. During job execution, input values are visible
as exported shell variables. For example:

```yaml
inputs:
  reference_file: gs//references/my_file
  main_file:
    - input1
    - input2
```

In all jobs launched by this pipeline, the shell variables `$reference_file` and
`$main_file` would be exported and point to a valid local filepath for the inputs.
See more about file localization below

## resources

The `resources` section (`--resources resourceName:value`) is used to specify
additional arguments to SLURM when launching the pipeline. Each _resourceName_ should
refer to a specific commandline option to SBATCH. For a full list of acceptable options,
see the [SBATCH](https://slurm.schedmd.com/sbatch.html) man pages. Each _resourceName_
in this section should match either a short or long form command option to sbatch,
without any leading dashes (set a value to `true` for options which take no arguments).
For example:

```yaml
resources:
  contiguous: true
  constraint: intel&gpu
  cpus-per-task: 2
```
would result in these command line arguments being passed: `sbatch --contiguous --constraint "intel&gpu" --cpus-per-task 2`
in addition to other options set by canine.

## adapter

The `adapter` section (`--adapter varname:value`) specifies which type of input
adapter to use, and how it should be configured. Adapters are used to parse the
input configuration provided by the user into a specific set of inputs for each
job. Below are the two types of available input adapters, and an example configuration for each.

### Manual (default) adapter

The Manual adapter is default. It takes inputs at face value. Constant inputs will
have the same value in every job. Array inputs will take one value per job. You can
also specify how multiple array inputs should be combined, by setting the `adapter.product` key.
By default, all array inputs are expected to be of equal length, and one job will
be launched for every value of the arrays. If `adapter.product` is enabled, arrays
may be of different lengths, and each job will be launched with a unique combination
of array inputs. The manual adapter does not do any handling of outputs besides
base delocalization.

Here is an example adapter configuration using the default settings:
```yaml
adapter:
  type: Manual
  product: false
```

Here is the equivalent command line options:
```
--adapter type:Manual --adapter product:false
```

### Firecloud adapter

The Firecloud adapter interprets inputs as Firecloud expressions. Each input
is evaluated in the context of entities from a given workspace. No inputs can be provided by the
user as an array. The user must provide a workspace, entity type, and entity name.
In that case, each input is evaluated as an expression in the context of that entity,
and only one job will be launched, using the results of evaluating the expressions.
If an entity expression is provided, one job will be launched for each entity resulting
from the given expression and inputs will be evaluated once each.

#### Configuration

To use the Firecloud adapter, you must specify `adapter.type: Firecloud` (`--adapter type:Firecloud` on the commandline).
You must also set the following required options:
* `adapter.workspace` (`--adapter workspace:name`) to be a FireCloud workspace in
the format `namespace/workspace`.
* `adapter.entityType` (`--adapter entityType:etype`) to be the type of entity
that the `entityName` key refers to
* `adapter.entityName` (`--adapter entityName:name`) to be the name of an entity
in the workspace, with the provided type

Optional configuration:
* `adapter.entityExpression` (`--adapter entityExpression:expr`) to be an expression
which maps the given entity to one or more entities that jobs should be run on.
This is the only way to launch more than one job with this adapter
* `adapter.write_to_workspace` (`--adapter write_to_workspace:true`) (defaults to True).
If enabled, job outputs will be written back to the FireCloud workspace using the
given output names (see below) as attributes

Here is an example adapter configuration using a fake workspace:

```yaml
adapter:
  type: Firecloud
  workspace: example-namespace/example-workspace
  entityType: sample_set
  entityName: all_samples
  entityExpression: this.samples
  write_to_workspace: true
```
Here is the equivalent command line options:
```
--adapter type:Firecloud --adapter workspace:example-namespace/example-workspace --adapter entityType:sample_set --adapter entityName:all_samples --adapter entityExpression:this.samples --adapter write_to_workspace:true
```

## backend

The `backend` section (`--backend varname:value`) configures the SLURM backend that
should be used to launch jobs. Backends are used to launch and monitor jobs, as well
as to localize inputs and delocalize outputs. Below are the 3 types of Backends,
and an example configuration for each

### Global options

This section lists backend options which can be applied to all backends
* `type`: Specifies the backend type (`Local`, `Remote`, or `TransientGCP`)
* `slurm_conf_path`: Specifies the path to the `slurm.conf` path.
* `hard_reset_on_orch_init`: If this is `True` and `slurm_conf_path` is provided,
`slurmctld` will be halted and reconfigured using this path before the job is submitted.
This is useful for `Local` and `Remote` backends to fix corrupted slurmctl.
**Note:** This path must be valid within the slurm controller. It will not be localized
from the current filesystem to the slurm controller

### Local (default) backend

This backend is the default, and runs SLURM commands through the local system.
This only works if your computer is a validly configured SLURM controller node, i.e.:

* slurmctld, munged, and slurmdbd run properly
* Accounting is enabled (i.e., `sacct` can list completed jobs)

This is the default, so no configuration is required, but here is an example of
how to explicitly set the backend:

```yaml
backend:
  type: Local
```

Here is the equivalent command line option:
```
--backend type:Local
```

**Warning:** Do not specify `localization.transfer_bucket` when using a `LocalSlurmBackend`.
This may result in redundant gsutil file transfers. Without a transfer bucket defined,
a `LocalSlurmBackend` will use file copies and symlinks to stage inputs, even though
a message about "SFTP" is displayed

### Remote backend

This backend is used to SSH to a SLURM controller or login node and dispatch jobs
there. File localization and delocalization happens over SFTP. The only required
option is `backend.hostname` which specifies the remote host to connect to. For a
full list of options to this backend, see the [paramiko.SSHClient.connect](http://docs.paramiko.org/en/2.5/api/client.html#paramiko.client.SSHClient.connect) docs. Each argument to that function
may be specified as an option to `backend`.

**Note:** Canine will attempt to read your ssh config and translate it into arguments
that paramiko can understand. The following ssh config options are currently supported
(more options will be added soon):

* `hostname`: Changes the actual hostname passed to `connect`
* `port`: Changes the port from the default 22
* `user`: Changes the username passed to `connect`
* `identityfile`: Changes the key_filename passed to `connect`
* `userknownhostsfile`: Additionally loads known hosts from the provided filename
* `hostkeyalias`: Uses the known host key from the given alias instead of the given
hostname. **Warning:** If the `hostkeyalias` is not found in the known hosts file,
then known hosts will be ignored; the backend will connect without checking the remote
ssh fingerprint

Here is an example Remote configuration

```yaml
backend:
  type: Remote
  hostname: slurm-controller
  user: my-user
  identityfile: ~/.ssh/slurm_id_rsa
```

Here is the equivalent command line options:
```
--backend type:Remote --backend hostname:slurm-controller --backend user:my-user --backend identityfile:~/.ssh/slurm_id_rsa
```

### TransientGCP backend

This backend is used to create a new Ubuntu 18.04 SLURM cluster on GCP, then shut it down after
running the pipeline. It requires that you have the Google Cloud SDK installed.
Depending on your cluster configuration, it may take up to 10 minutes before Canine
begins localizing files to the server and another 10 before the cluster is ready
to accept jobs. Here is a list of available options for this backend:

* `name`: The name to use for resources created by the cluster in GCP (defalt: slurm-canine)
* `max_node_count`: The maximum number of compute nodes to have running at any time (default: 10)
* `compute_zone`: The [Compute Zone](https://cloud.google.com/compute/docs/regions-zones/)
in which to create resources (default: us-central1-a)
* `controller_type`: The [Compute Instance Type](https://cloud.google.com/compute/pricing#predefined)
to use for the controller node (default: n1-standard-16)
* `worker_type`: The [Compute Instance Type](https://cloud.google.com/compute/pricing#predefined)
to use for the compute nodes (default: n1-highcpu-2)
* `compute_disk_size`: The size of the disk for each compute node, in gigabytes (default: 20)
* `controller_disk_size`: The size of the controller node's disk, in gigabytes (default: 200; Max: 2000)
* `secondary_disk_size`: The of a secondary disk (default: 0; Max: 64000). If you
use the secondary disk, it is mounted at `/mnt/disks/sec`. Make sure you set
`localization.staging_dir` to be a path within this directory.
* `gpu_type`: The [type of GPU](https://cloud.google.com/compute/pricing#gpus) to
attach to compute nodes (default: No gpus)
* `gpu_count`: The number of gpus to attach to each compute node (default: No gpus).
If any gpus are attached, nvidia drivers and the nvidia docker runtime will be
automatically installed on the compute nodes
* `project` : The Google Cloud Project to use when creating the Slurm cluster
* `controller_script`: Additional commands to run during setup of the SLURM controller node
* `compute_script`: Additional commands to run during setup of the SLURM compute node image

**Note:** By default, the Slurm cluster is created using your default Google Cloud
Project. However, you can explicitly set a project with `project`,
and that value will be used instead. You must have the following permissions on the
project to use it for the TransientGCP backend:
  * deploymentmanager.deployments.{create,get,stop,update,delete}
  * compute.disks.{create,get,delete,use}
  * compute.images.{create,delete,get}
  * compute.instances.{create,delete,get,list,osAdminLogin,setMetadata,start,stop,suspend,update}
  * compute.subnetworks.update
  * compute.routers.{create,delete,get,use,update}

**Note:** The controller node will also act as an NFS server which shares the `/home`
and `/apps` directories with the compute nodes. For this reason, the controller is
given a large disk (as most job data will reside there) and a powerful instance type
(to avoid becoming a bottleneck to the compute nodes).

**Note:** The TransientGCP backend is configured to schedule jobs as if each hyper-thread
were an actual physical core. This allows applications to access all available vCPUs
on the compute nodes. If your application does not support hyper-threading, this
will cause massive overutilization of the CPUs. In this case, you must either
adjust your `resources` to request _double_ the cpus that you will actually use
or use your own slurm cluster (with the `Remote` or `Local` backends) and configure
the nodes, and SelectTypeParameters such that multiple jobs cannot share the same
core

This backend can run without configuation, but here is an example:

```yaml
backend:
  type: TransientGCP
  name: slurm-canine-example
  controller_type: n1-standard-2
  worker_type: n1-standard-1
  controller_disk_size: 50
```

Here is the equivalent command line options:
```
--backend type:TransientGCP --backend name:slurm-canine-example --backend controller_type:n1-standard-2 --backend worker_type:n1-standard-1 --backend controller_disk_size:50
```

### TransientImage backend (alpha)

This backend is similar to LocalBackend, but it does not assume that the Slurm
cluster is already running. Rather, it takes a user-provided GCE image and
dynamically spins up nodes from this image.

This backend assumes:
* As with the local backend, the current node is a valid Slurm controller, i.e.:
    * `slurmctld`, `munged`, and `slurmdbd` run properly (note that these do not need
      to already be running when invoking Canine with this backend; they will
      be started automatically as needed.)
    * Accounting is enabled (i.e., `sacct` can list completed jobs)
* The default Slurm partition is compatible with any nodes that get spun up:
    * Worker node names must match names specified in a partition defined in `slurm.conf`
    * Worker node types must be consistent with node definitions in `slurm.conf`
* The image provided has a valid Slurm installation, compatible with that of the
  controller node (e.g., same version, same plugins, etc.)
* Slurm configuration files are present at a path accessible by all nodes (e.g.,
  an NFS server, or mirrored across all nodes)
* If GPUs are added, drivers must already be installed

Configuration options are similar to those of the TransientGCP backend; the following
options are identical:

* `compute_zone`: The [Compute Zone](https://cloud.google.com/compute/docs/regions-zones/)
in which to create resources (default: us-central1-a)
* `worker_type`: The [Compute Instance Type](https://cloud.google.com/compute/pricing#predefined)
to use for the compute nodes (default: n1-highcpu-2). **This must match the node specification
defined in `slurm.conf`.**
* `secondary_disk_size`: The of a secondary disk (default: 0; Max: 64000). If you
use the secondary disk, it is mounted at `/mnt/disks/sec`. Make sure you set
`localization.staging_dir` to be a path within this directory. (**not yet implemented
in alpha release**)
* `gpu_type`: The [type of GPU](https://cloud.google.com/compute/pricing#gpus) to
attach to compute nodes (default: No gpus) (**not yet implemented in alpha release**)
* `gpu_count`: The number of gpus to attach to each compute node (default: No gpus).
If any gpus are attached, nvidia drivers and the nvidia docker runtime will be
automatically installed on the compute nodes (**not yet implemented in alpha release**)
* `project` : The Google Cloud Project to use when creating the Slurm cluster.
(default: current working project)
* `controller_script`: Additional commands to run during setup of the SLURM controller node
(**not yet implemented in alpha release**)
* `compute_script`: Additional commands to run during setup of the SLURM compute node image

The following options are unique to the TransientImage backend:

* `image`: Name of GCE image to instantiate worker nodes with. Must exist in
current project. **Mandatory.**
* `slurm_conf_path`: Path to `slurm.conf`. Must be exist on both controller
node and all worker nodes. **Mandatory.**
* `worker_prefix`: Prefix for node names (e.g., slurm_canine1 ... slurm_canine50).
(default: slurm_canine). **Must match the partition specification in `slurm.conf`.**
* `tot_node_count`: Total number of nodes to create. (default: 50)
* `init_node_count`: Initial number of nodes to start. (default: `tot_node_count`)
* `preemptible`: Whether the nodes to create are
[preemptible](https://cloud.google.com/preemptible-vms/). (default: True)
* `user`: User under which Slurm controller is launched. (default: root)
* `delete_on_stop`: Whether to delete worker nodes entirely after Canine exist,
or just shut them down. (default: False)

Here is an example minimal configuration file:

```yaml
backend:
  type: TransientImage
  image: my_cluster_image
  slurm_conf_path: /nfs/slurm/conf/slurm.conf
```

## localization

The `localization` section (`--localization varname:value`) specifies options for
how input data is transferred and made available to the SLURM cluster. Here is a
description of the options for Localization:

* `common`: If True, any input files which appear more than once anywhere in the
job configuration will be localized once to a common directory instead of being
localized multiple times into each job's input directories (default: True)
* `staging_dir`: The directory on the SLURM controller where job inputs should be
localized, and where jobs should be staged (default: create random directory in user's home).
This assumes that the controller and compute nodes are connected via NFS or other
network filesystem and that the home folder is shared. Always set this such that
the staging_dir is part of a shared volume. If no such share exists, set to `SBCAST`
to use slurm's `sbcast` command to copy job inputs.
**Warning**: `mount_path` has been removed as an option. All localizers now assume that
the controller and compute nodes have a shared NFS mounted at `staging_dir`
* `project` : The Google Cloud Project to charge when interacting with requester pays buckets.
If the `transfer_bucket` or any declared inputs are requester pays, this project will
be billed for the charges
* `transfer_bucket`: A Google Cloud Storage bucket to use when executing batch directory
transfers between the slurm cluster and the local filesystem. Providing a `transfer_bucket`
_vastly_ improves file transfer performance when copying directories, however there
are some considerations:
    * **Do not** include `gs://` or any other path in the bucket. Transfers will
    be executed through temporary directories which are cleaned up afterwards.
    For example, if you want to transfer using `gs://my_bucket`, simply use the
    bucket name: `my_bucket`
    * **Do not** use a nearline or coldline bucket for transfers. You will incur
    massive storage fees
    * **Do not** use `transfer_bucket` with the `Local` backend. Doing so will
    result in unecessary file transfers
    * Using a transfer bucket does not preserve empty directories. To preserve
    directory structure, either add an empty file to each directory and subdirectory
    you wish to transfer, or do not set a `transfer_bucket`. The fallback SFTP system
    is significantly slower, but will preserve directory structure
    * Directories containing directory symlinks will not be preserved. It's okay
    if the starting directory is a symlink but only file symlinks will be followed after that.
    Do not set a `transfer_bucket` if you wish to preserve the **apparent** structure,
    meaning all symlinks will be resolved during the transfer
* `strategy`: The localization strategy to use. Options:
    * `Batched` (default): Localization takes place on the local filesystem, then
    is transferred to the remote cluster near the end of localization. Gsutil files
    are localized on the remote system during this same finalization step.
    * `Local`: Same as the default `Batched` strategy, except that Gsutil files
    are localized to the local staging directory and transferred to the remote
    cluster along with the rest of localization files
    * `Remote`: Localization takes place entirely on the remote cluster
    * `NFS`: Localization takes place entirely local, and assumes that an NFS share
    will ensure the data is localized/delocalized. See `NFS` section below

**NOTE:** The old `localizeGS` option has been removed. From now on,
if you do not wish to automatically localize `gs://` paths, use an appropriate override

### overrides

The localization section also allows for individual overrides of input handling.
Overrides can be specified in the `localization.overrides` section or with `--localization overrides:inputName:value`.
In this section, you can override default localization handling for a given input
by specifying an override value for the input name (for instance, an input named `foo`
  would be overridden with `--localization overrides:foo:value`). The default handling
for any given input is as follows:

* If the file appears as an input to multiple jobs (for instance, if it was a constant input)
and `localization.common` is enabled, the file will be localized to the `$CANINE_COMMON`
directory instead of the job's input. Actual handling of the file follows the remaining rules:
* If the file starts with `gs://`, the file
will be treated as a Google Storage file and will be copied by invoking `gsutil cp`
though the current backend
* If the file is a valid file path on the local system, it will be copied using
the current backend's transport system (SFTP for Remote and TransientGCP backends)
* Otherwise, the input is treated as a regular string and will not be localized.

Here is a list of the different override types, and their function:

* `Stream`: Instead of copying the whole file to the remote system, the file will
be streamed into the job via a named pipe. The input's environment variable will
point to the pipe. This only works for `gs://` files, and will override default
common behavior (file will always be streamed to job-specific input directory)
**Warning:** Streams are included as part of a job's resource allocation. Having too
many streamed files may adversely affect job performance as gsutil competes with
the main job script for the CPU. If you stream more than ~3 files per job, consider increasing
`resources.cpus-per-task`
* `Localize`: Force the input to be localized. This will override default
common behavior (file will always be localized to job-specific input directory).
If the input is neither a `gs://` path nor valid local filepath, an exception will be raised
* `Delayed`: Instead of copying the whole file during the setup phase (before any jobs
  are queued on the SLURM cluster), the file will be localized as part of each individual
job's script. This only works for `gs://` files, and will override default
common behavior (file will always be localized to job-specific input directory).
The file will only be localized once. If the job is restarted (for instance, if the
  node was preempted) the file will only be re-downloaded if the download did not
  already finish
* `Common`: Forces the input to be localized to the common directory. This will
override default common behavior (the file will always be localized to the `$CANINE_COMMON` directory)
* `null`: Forces the input to be treated as a plain string. No handling whatsoever
will be applied to the input.

### NFS Localizer

This localizer assumes that both the current system, the slurm controller, and slurm
compute nodes are all linked by at least one common NFS share. Please read the following
notes when using the NFS localizer:

* The `transfer_bucket` option has no effect. Data is never actively transferred,
only passively over NFS
* The `staging_dir` option refers to the path where canine should be staged **on the local system**.
This path _must_ exist within the NFS share
* `staging_dir` is a required option for the NFS strategy
* The `mount_path` option refers to the path where the staging directory will be visible
**on the remote systems**. The slurm controller and worker nodes must all use this path

### Google Cloud Storage

Localization uses credentials on the remote server to localize `gs://` files.
Files localized during job setup (default, `Common`, `Localize`) will use the credentials
of the SLURM node your backend is connected to. However, `Stream` and `Delayed`
files are handled as part of a job's script and will use credentials of the compute
node. If you wish to use your current credentials to localize default, `Common`, and `Localize`
files, set your localization strategy to `Local`

Here is an example configuration for localization:
```yaml
inputs: # from example above
  reference_file: gs://references/my_file
  main_file:
    - input1
    - input2
localization:
  common: true
  staging_dir: ~/my-job
  overrides:
    reference_file: Common
    main_file: Delayed
```

Here is the equivalent command line options:
```
--localization common:True --localization staging_dir:'~/my-job' --localization overrides:reference_file:Common --localization overrides:main_file:Delayed
```

## outputs

The `outputs` section (`--outputs outputName:pattern`) is used to specify which
files will be delocalized from the SLURM cluster (regardless of if the job succeeded or not).
Each outputName should be a file glob pattern, relative to each job's initial CWD
(`$CANINE_JOB_ROOT`). All files matching a pattern will be delocalized. Files are
delocalized to your filesystem in a `canine_output` folder in your current directory:

```
canine_output/
  {jobId}/
    {outputName}/
      ...files matching outputName's pattern...
```

By default, a job's stdout and stderr streams are delocalized, but you can override
this by specifying custom patterns for `stdout` and `stderr`, respectively.

Here is an example output configuration:

```yaml
outputs:
  counts: *.counts.txt
  results: *.tar.gz
```

Here is the equivalent command line options:
```
--outputs counts:"*.counts.txt" --outputs results:"*.tar.gz"
```

---

## Job Environment Variables

In addition to all variables present during [SLURM batch jobs](https://slurm.schedmd.com/sbatch.html#lbAI)
and all variables provided based on config inputs, Canine also exports the following
variables to all jobs:
* `CANINE`: The current canine version
* `CANINE_BACKEND`: The name of the current backend type
* `CANINE_ADAPTER`: The name of the current adapter type
* `CANINE_ROOT`: The path to the staging directory
* `CANINE_COMMON`: The path to the directory where common files are localized
* `CANINE_OUTPUT`: The path to the directory where job outputs will be staged during delocalization
* `CANINE_JOBS`: The path to the directory which contains subdirectories for each job's inputs and workspace
* `CANINE_JOB_VARS`: A colon separated list of the names of all variables generated by job inputs
* `CANINE_JOB_INPUTS`: The path to the directory where job inputs are localized
* `CANINE_JOB_ROOT`: The path to the working directory for the job. Equal to CWD at the start of the job. Output files should be written here
* `CANINE_JOB_SETUP`: The path to the setup script which ran during job start
* `CANINE_JOB_TEARDOWN`: The path the the teardown script which will run after the job
