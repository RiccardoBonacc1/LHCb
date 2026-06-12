# Nectar computing cluster
The group has access to a computing cluster for performing simulations and analysis of LHCb data. It is configured to be as identical as possible to the `lxplus` machines at CERN. It is a linux cluster that you access via `ssh`.

In general, anything you read in the [StarterKit](https://lhcb-starterkit-run3.docs.cern.ch/) to do on `lxplus` can be done on the `nectar` cluster as well.

## Obtain an account
Logins to the cluster can only happen via using an `ssh` keypair. This is similar to the passkey methods that are used on many websites. On your Windows/Mac/linux laptop, open a terminal (command) window. In that one type `ssh-keygen`. Accept the default name and then provide a password that is at least 12 characters long. The program will report that it writes two files into the hidden `.ssh` folder below your home folder. Via either email or Slack send the content of the files ending in `.pub` to your supervisor who will then forward it to Ulrik.

Wait for Ulrik to tell you that the account is ready and give you a username. Then create (or edit if already there) a file called `config` in the hidden `.ssh` folder inside your home folder. In it put the content
```sshconfig
Host nectar9
     User <user>
     PubkeyAuthentication yes
     IdentityFile ~/.ssh/<keyfile>
     ProxyJump frontend

Host frontend
     User <user>
     PubkeyAuthentication yes
     IdentityFile ~/.ssh/<keyfile>
     Hostname frontend.lhcb-simulation.cloud.edu.au
     ControlPath /run/user/%i/%r@%h:%p
     ControlMaster auto
```
where you replace `<user>` with the username you are given and `<keyfile>` is the name of the file that doesn't have the `.pub` extension. 

> **Note:**
> If you connect from a Windows machine, replace the directory in `ControlPath` with `~/.ssh/sockets/%r@%h-%p` and make sure to create the folder `sockets` inside your hidden `.ssh` folder. If it still doesn't work, try to completely remove the `ControlPath` and `ControlMaster` lines.

## Setup two factor authentication
Contact Ulrik to setup two factor authentication. In most cases this requires a visit to his office.

## Test account
Now from the terminal (command) window try
```bash
ssh nectar9
```
and confirm that it logs you in. If it doesn't work, do 
```bash
ssh -vvv nectar9
```
and ask for help, including the output of the last command.

## Disk space
On the `nectar` cluster you have a home directory at `/home/<user>`. You are the only one who can read content in that directory. To share data with other users, create a directory below `/shared` that has an explaining name. Everybody on the cluster will have access to those files.

Be aware that there is **no backup** and **no undo** of files you delete, or indeed everything if there is a disk failure. For code, you should use GitHub/Gitlab or similar to track your changes and for data, it should either be available elsewhere or should be relatively easy to recreate.

## Using the Nectar computing cluster with HTCondor

HTCondor is a system for assigning independent executables or "jobs" across shared computational resources, which is perfect in this case, as we can use it to manage jobs on a large scale executed on the Nectar machine. The Nectar cluster has a portion of its computational resources for the tasks we perform day-to-day, referred to as the 'frontend' (the `nectar9` machine you log in to); this has 16 CPU cores and 62 GB of RAM. The other portion of the cluster consists of large worker nodes, referred to as the 'backend', with 32 CPU cores and 128 GB of RAM each. The number of workers can vary as they are provisioned on demand, but all together they pool to roughly 390 cores and 1.5 TB of RAM. The worker nodes are purely used to execute jobs on a large scale, quickly, through parallelisation. Note that the workers only have around 20 GB of local scratch disk each, so jobs should write their outputs back to `/home` or `/shared` rather than accumulate data on the worker itself.

A quick start guide can be found [here](https://htcondor.readthedocs.io/en/25.0/users-manual/quick-start-guide.html), there is also some specific particle physics use cases below.

### Running an executable on a large group of data files

One example of such a task is applying an MVA decision to all your different ROOT data files. For all HTCondor pipelines, you are running a bash script with arguments, where each job executes the script independently with its own set of arguments. Such a script can be named `apply_run.sh`, and is shown below:

```
#!/bin/bash

# Create MVA terms  

FOLD=$1
NUMBER=$2
POLARITY=$3
YEAR=$4
MC=$5

# Activate the base environment
source /home/bonacci/miniconda3/bin/activate base
conda activate env

cd /home/bonacci/xiccmultiplicity/ApplySelection/Step3_MVA_Training/Lc2pKPi

# Run MVA classifier on Lc data:
python apply.py -f $FOLD -n $NUMBER -p $POLARITY -y $YEAR -mc $MC

echo "Done"

```

In the bash script you can house things like running a python script (here `apply.py`) with arguments that differ from job to job. Keep in mind that the script runs on a worker node, not in your interactive shell, so it must set up its own environment -> source your conda/LbEnv environment and `cd` to the directory where your scripts live, as in the example above (replace the paths with your own).

You will then have to set up the parameters that are inserted as the arguments. This is done in an HTCondor *submit description file*. Note that this is **not** a bash script, it has its own syntax, and by convention it is given the extension `.sub`. Such a file (`apply_submit.sub`) is shown below:

```
executable            = apply_run.sh
output                = logs/apply_run.$(ClusterId).$(ProcId).out
error                 = logs/apply_run.$(ClusterId).$(ProcId).err
log                   = logs/apply_run.log

# Insert a job name here
+JobBatchName         = "Apply MVA Data/MC"

# Set the CPU and memory requirements of each job
request_cpus          = 1
request_memory        = 2000M

# Stream output/error live while the job runs
stream_output         = True
stream_error          = True

# Run over folds, numbers, polarities, years, MC/data:
# Queue rows are: 
# FOLD=$1
# NUMBER=$2
# POLARITY=$3
# YEAR=$4
# MC=$5

arguments             = $(FOLD) $(NUMBER) $(POLARITY) $(YEAR) $(MC)
QUEUE_FILE            = queue_args.txt
queue FOLD, NUMBER, POLARITY, YEAR, MC from $(QUEUE_FILE)

```

The first line gives the executable, which in this case is the bash script you want to run. The corresponding `output`, `error` and `log` lines say where each job's stdout, stderr and the overall HTCondor event log are written; the log file is useful in figuring out what is happening (or has happened) to your jobs. Note that HTCondor will **not** create the `logs` directory for you — create it before submitting with `mkdir -p logs`, otherwise the jobs will fail to start. The `request_cpus` and `request_memory` lines do as they sound, setting the number of CPUs and the memory limit for each job; the memory is something to gauge depending on the jobs being executed, and a job that exceeds its request will be put on hold. Then there are the `arguments`, which must be in the same order as the bash script expects them -> this is where the per-job arguments are filled in. The `QUEUE_FILE` is a text file that you must create, housing the argument values for each job, one job per line. An example of `queue_args.txt` for the above is displayed below:

```
0 0 MagDown 2018 0 
0 0 MagUp 2018 0 
0 1 MagDown 2018 0 
0 1 MagUp 2018 0 
0 2 MagDown 2018 0 
0 2 MagUp 2018 0 
0 3 MagDown 2018 0 
0 3 MagUp 2018 0 
0 4 MagDown 2018 0 
0 4 MagUp 2018 0 
...
```

The last line of the submit file queues one job per row of the queue file, with the corresponding arguments. We can then finally submit the jobs to the batch system through the command:

```
condor_submit apply_submit.sub
```

Now our jobs will all be submitted and you should get back something like:

```
[bonacci@nectar9 Lc2pKPi]$ condor_submit apply_submit.sub 
Submitting job(s)..................................................................................................................................................................................................................................................................................................................................................................................................................
402 job(s) submitted to cluster 10035.
```

There are different commands to check on the status of jobs, though the main one is `condor_q`. Doing so will output a summary of the status of all jobs submitted:

```
[bonacci@nectar9 Lc2pKPi]$ condor_q


-- Schedd: nectar9.novalocal : <192.168.2.134:9618?... @ 06/11/26 17:43:38
OWNER   BATCH_NAME                      SUBMITTED   DONE   RUN    IDLE  TOTAL JOB_IDS
bonacci Apply MVA Data/MC              6/11 17:43      _      1    401    402 10035.0-401

Total for query: 402 jobs; 0 completed, 0 removed, 401 idle, 1 running, 0 held, 0 suspended 
Total for bonacci: 402 jobs; 0 completed, 0 removed, 401 idle, 1 running, 0 held, 0 suspended 
Total for all users: 402 jobs; 0 completed, 0 removed,
```

You can of course also check on the status of jobs by looking in `logs` to see what the jobs are outputting. This is where `stream_output` and `stream_error` are particularly nice, as you are able to see the output while the jobs run, which could save you time if you notice something going wrong while a job is still running. If these are not set, the job output and error files only appear after a job has completed. Submitted jobs will sit in `IDLE` until they are assigned to a worker node, at which point they will `RUN`. If a problem is encountered in the script, or perhaps the memory constraint is reached, the job will be `HELD`. In this case `condor_q -held` shows the hold reason directly, and the `logs` usually contain the details. Jobs can be removed with `condor_rm <ClusterId>` for the whole batch, `condor_rm <ClusterId>.<ProcId>` for a single job, or `condor_rm $USER` for everything you have submitted. As jobs finish they move to the `DONE` column, and once all jobs in the cluster have completed they disappear from `condor_q` entirely, you can still inspect finished jobs with `condor_history` (or `condor_history -l <jobid>` for full details). If everything has gone through, you have successfully executed all your jobs.

### Downloading a group of data files from Analysis Productions (any files from DIRAC)

Like the example above, downloading a large group of files from the grid parallelises perfectly with HTCondor: one job per file. The extra ingredient compared to the previous example is that reading files from grid storage (PFNs of the form `root://eoslhcb.cern.ch//eos/lhcb/...`) requires a valid **grid proxy**, and that proxy has to be made available to every job on the worker nodes.

#### Setting up the grid proxy

First, generate a proxy on the frontend:

```bash
lhcb-proxy-init
```

By default the proxy is written to `/tmp/x509up_u$(id -u)`. It is however much better to export it to a fixed location in your home directory, by setting the `X509_USER_PROXY` environment variable in your `.bashrc` **before** running `lhcb-proxy-init`, both the proxy creation and all grid tools respect it:

```bash
export X509_USER_PROXY=/home/<user>/.grid.proxy
```

The reason this matters: a proxy is only valid for **24 hours**. If your batch of download jobs takes longer than that (hundreds of large files can easily run over several days as jobs queue up), jobs that start after the proxy expires will fail to authenticate. With the proxy at a fixed path in your home directory, you can simply re-run `lhcb-proxy-init` at any point (i.e. once in every 24-hour period) and the renewed proxy is picked up from the same path, allowing the remaining jobs to continue without resubmitting anything. Check what is left on your proxy at any time with `lhcb-proxy-info`.

#### Getting the list of PFNs

Next you need the list of PFNs to download. For files from [Analysis Productions](https://lhcb-productions.web.cern.ch/ana-prod/) the easiest way is the [apd](https://gitlab.cern.ch/lhcb-dpa/analysis-productions/apd) python package, with which you can look up your production and write the PFNs into the HTCondor queue file. A script (`Load_pfn_names.py`) for doing this is shown below:

```python
from apd import AnalysisData

# The working group and analysis name of your production:
datasets = AnalysisData("charm", "xicc_24mctrack")

job_names = (
    "mc_24_w31_magup_sim10g-redecay01_26266052_xiccpp2lckpipi_tuple",
    "mc_24_w32_34_magup_sim10g-redecay01_26266052_xiccpp2lckpipi_tuple",
)

entries = []
for name in job_names:
    pfns = datasets(config="mc", datatype="2024", filetype="tuple.root",
                    polarity="magup", eventtype="26266052", name=name)
    for pfn in pfns:
        entries.append((name, "Hcc_LcpKmPipPip", pfn, "MC"))

# Write the queue file: JOB_NAME JOB_TREE IDX TOTAL PFN DATATYPE
total = len(entries)
with open("condor_pfn_queue.txt", "w") as f:
    for idx, (job_name, tree, pfn, datatype) in enumerate(entries):
        f.write(f"{job_name} {tree} {idx} {total} {pfn} {datatype}\n")

print(f"Written {total} entries to condor_pfn_queue.txt")
```

The arguments to the `datasets(...)` call select which sample you want (data taking year, polarity, event type etc.), you can find the right values for your production on the Analysis Productions web page. Each row of the resulting `condor_pfn_queue.txt` contains everything one job needs: a name and tree for the output file, an index so the output files are uniquely named, the PFN to read, and a flag for whether it is data or MC.

#### The submit file

The submit file (`download_pfns_submit.sub`) follows the same pattern as before, with one important addition: the `use_x509userproxy` lines tell HTCondor to ship your grid proxy into each job's sandbox and set `X509_USER_PROXY` for the job automatically.

```
universe              = vanilla
executable            = download_pfns_run.sh

# Queue file to read. This can be overridden at submit time
# (`condor_submit download_pfns_submit.sub QUEUE_FILE=retry_queue.txt`),
# which is handy for resubmitting only the jobs that failed.
QUEUE_FILE            = condor_pfn_queue.txt

# Queue rows are: JOB_NAME JOB_TREE IDX TOTAL PFN DATATYPE
arguments             = $(JOB_NAME) $(JOB_TREE) $(IDX) $(TOTAL) $(PFN) $(DATATYPE)

output                = condor_logs/download_pfns.$(ClusterId).$(ProcId).out
error                 = condor_logs/download_pfns.$(ClusterId).$(ProcId).err
log                   = condor_logs/download_pfns.log
+JobBatchName         = "DownloadPFNs"

# Resource requests: ROOT jobs exceed the default 128MB limit.
request_cpus          = 1
request_memory        = 2GB
request_disk          = 200MB

getenv                = True
stream_output         = True
stream_error          = True

# XRootD access to eoslhcb requires a valid grid proxy.
use_x509userproxy     = True
x509userproxy         = $ENV(X509_USER_PROXY)

queue JOB_NAME, JOB_TREE, IDX, TOTAL, PFN, DATATYPE from $(QUEUE_FILE)
```

#### The job scripts

The executable bash script (`download_pfns_run.sh`) is again the same pattern as before: take the arguments, set up the environment, work out the output path and run the python script that does the actual download. Write the output somewhere with enough space, such as a directory under `/shared` (remember the worker nodes themselves only have ~20 GB of scratch disk).

```bash
#!/bin/bash

JOB_NAME="$1"
JOB_TREE="$2"
IDX="$3"
TOTAL="$4"
INPUT_FILENAME="$5"
DATATYPE="$6"

source /home/bonacci/miniconda3/bin/activate base
conda activate env

if [[ -z "${X509_USER_PROXY:-}" || ! -r "${X509_USER_PROXY}" ]]; then
    echo "ERROR: no readable X509 proxy found" >&2
    exit 3
fi

OUTPUT_DIR="/shared/XiccMultiplicityRun3/Xicc2LcKPiPi/${DATATYPE}"
OUTPUT_PATH="$OUTPUT_DIR/${JOB_NAME}_${JOB_TREE}_${IDX}of${TOTAL}.root"

echo "PFN: $INPUT_FILENAME | Tree: $JOB_TREE | $IDX/$TOTAL -> $OUTPUT_PATH"

python -u download_pfns.py \
    -i "$INPUT_FILENAME" \
    -it "${JOB_TREE}/DecayTree" \
    -o "$OUTPUT_PATH" \
    -ot "${JOB_TREE}/DecayTree" \
    -d "$DATATYPE"

[[ -s "$OUTPUT_PATH" ]] || { echo "ERROR: output missing or empty: $OUTPUT_PATH" >&2; exit 5; }
echo "Done"
```

The check on the last line makes a failed transfer show up as a failed job, rather than a silently missing file.

If you just want an exact copy of each file, the python script can be replaced by a single `xrdcp "$INPUT_FILENAME" "$OUTPUT_PATH"`. Often though you want to skim the file while downloading it; select only certain trees, or apply a cut — so the files you store locally are much smaller. The python script (`download_pfns.py`) below streams the PFN with `RDataFrame`, applies a selection and saves only the result:

```python
"""Stream PFNs and write skimmed ROOT files."""

import argparse
import ROOT


def main(input_name, input_tree, output_name, output_tree, datatype):
    print(f"Processing file {input_name}")

    # Preflight open to catch authentication errors cleanly. If this fails,
    # it commonly means the grid proxy is missing or has expired.
    f = ROOT.TFile.Open(input_name)
    if not f or f.IsZombie():
        print(f"ERROR: failed to open input: {input_name}")
        raise SystemExit(10)
    f.Close()

    # Load the remote file as an RDataFrame:
    Data = ROOT.RDataFrame(input_tree, [input_name])

    # Apply a skimming selection while downloading, e.g. a loose MVA cut:
    if datatype == "lhcb":
        Data = Data.Filter("Hcc_MVA > 0.13")

    # Save a snapshot of the filtered data to the local output file:
    Data.Snapshot(output_tree, output_name)


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="PFN download/skim script")
    parser.add_argument("-i",  "--input_name",  type=str, required=True)
    parser.add_argument("-it", "--input_tree",  type=str, required=True)
    parser.add_argument("-o",  "--output_name", type=str, required=True)
    parser.add_argument("-ot", "--output_tree", type=str, required=True)
    parser.add_argument("-d",  "--datatype",    type=str, required=True)
    args = parser.parse_args()

    main(args.input_name, args.input_tree, args.output_name,
         args.output_tree, args.datatype)
```

#### Putting it all together

The whole pipeline can then be driven from a single steering script that checks the proxy, builds the queue file and submits:

```bash
#!/bin/bash
set -euo pipefail

# Ensure a readable grid proxy is available for XRootD access to eoslhcb.
if [[ -z "${X509_USER_PROXY:-}" || ! -r "${X509_USER_PROXY}" ]]; then
    echo "ERROR: No readable X509 proxy found. Run: lhcb-proxy-init" >&2
    exit 5
fi

# Build the queue file of PFNs to download:
python Load_pfn_names.py

# Submit the download jobs:
mkdir -p condor_logs
condor_submit download_pfns_submit.sub

# Block until all jobs in the batch have finished (useful when this
# is a step of a larger pipeline):
condor_wait condor_logs/download_pfns.log
```

One final practical tip: grid transfers occasionally fail for transient reasons (a storage element times out, a brief network glitch). Rather than resubmitting everything, copy the rows of the queue file corresponding to the failed jobs into a new file and resubmit only those with `condor_submit download_pfns_submit.sub QUEUE_FILE=retry_queue.txt`.