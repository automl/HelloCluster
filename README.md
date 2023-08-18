# Contents:
1. [Setting up the repo](#setting-up-the-repo)
2. [Running the code on KI-SLURM](#running-the-code-on-ki-slurm)
   - [Finding out partitions you have access to](#finding-out-partitions-you-have-access-to)
   - [Submitting jobs with sbatch](#submitting-jobs-with-sbatch)
   - [See your submitted jobs](#see-your-submitted-jobs)
   - [Cancel your submitted jobs](#cancel-your-submitted-jobs)
   - [See summary of the current status of the cluster](#see-summary-of-the-current-status-of-the-cluster)
   - [Running an interactive session](#running-an-interactive-session)
3. [Debugging on KI-SLURM](#debugging-on-ki-slurm)
   - [Installing Remote-SSH extension for VSCode](#installing-remote-ssh-extension-for-vscode)
   - [Debugging](#debugging)
4. [Running Jupyter Notebook on a GPU on the cluster](#running-jupyter-notebook-on-a-gpu-on-the-cluster)
   - [Making it easier](#making-it-easier)
6. [Cluster usage policy](#cluster-usage-policy)

# Setting up the repo
1. Clone this repository in the remote machine
2. Set up [Miniconda](https://docs.conda.io/en/latest/miniconda.html)
3. Create a new environment:
   *    `conda create --name hello_cluster_env python=3.9`
4. Activate the new environment and install the requirements:
   * `conda activate hello_cluster_env`
   * `pip install -r requirements.txt`
5. You should now be able to run the code as follows:
   * `python main.py --<cmd-line-arguments> <values>`

**IMPORTANT:** Remember, you are **not** supposed to run your python scripts this way on the clusters. Scripts should always be submitted as jobs (see next section). The *login nodes* should never be used for any kind of *compute* (not even, say, to run *Tensorboard*). Step 4 is only for local installations on your computer.

In the KI-SLURM (Meta) cluster, your account will be penalized by limiting your CPU usage for a while if you run compute-intensive processes on login nodes.

# Running the code on KI-SLURM

There are two ways to run scripts on the cluster:
1. Submit a job using `sbatch`
2. Run your code in a SLURM interactive session using `srun`

Let's begin by looking at `sbatch`. First, you need to know which partitions you have access to.

## Finding out partitions you have access to
You can use `sinfo` to find this information.  A sample output might look like this:

| PARTITION     |   AVAIL  | TIMELIMIT  | NODES | STATE | NODELIST      |
| ------------  |     ---  | ---------  | ----- | ----- | ------------- |
| partitionXYZ  |      up  | 2-00:00:00 |   1   | idle  | xyzgpu[0-6]   |
| partitionABC  |      up  | 01:00:00   |   1   | idle  | xyzgpu[10, 20]|

## Submitting jobs with sbatch
See `./scripts/meta/run.sh` for an example of a job script. To submit a job:
1.  Edit `run.sh`:
       * Add the partition you want to run the job on
       * Adjust the path to your miniconda installation
2.  Create the directories required for the logs
3.  `sbatch scripts/meta/run.sh`

## See your submitted jobs
   * `squeue` # See all the jobs in the queue
   * `squeue -u user`  # See only user's jobs

## Cancel your submitted jobs
   * `scancel -u my_user`  # Cancel all your jobs
   * `scancel <jobid>`  # Cancel a specific job


## See summary of the current status of the cluster
   * `sfree`
## Running an interactive session
You can run an interactive session using `srun`. You can specify the parameters of the job using the same switches seen in `run.sh`. You only require an additional `--pty bash` to start a bash session.

   * `srun --partition <your_partition> --mem 6GB --job-name HelloClusterInteractiveSession --pty bash`

You will see that you are now logged into a compute node. From here, you may run python scripts as usual:
   * `python main.py --device cuda`

*Remember that you should only do this from a *compute node* that you acquired using `srun`, never a login node.*

# Debugging on KI-SLURM
To remotely debug code residing in the cluster, we require the `Remote-SSH` extension for [VSCode](https://code.visualstudio.com/). We shall begin by installing it.

## Installing Remote-SSH extension for VSCode

Install the `Remote-SSH` extension for VSCode as follows:

1. Bring up the Extensions view (Ctrl+Shift+X / Cmd+Shift+X). Or, `View` > `Extensions`
2. Install `Remote - SSH` extension (Extension ID: ms-vscode-remote.remote-ssh)

## Debugging
You can run your code that resides on the cluster while debugging it with VSCode as follows:
1. Start an interactive session:
   * `srun -p <partition_name> --pty bash <...other options>`
2. This will log you into a node with the requested resources
   ```myuser@dlcgpuxyz:~$```
   Here, `myuser` is logged into node `dlcgpuxyz`
4. Now, use the `Remote-SSH` extension to log into that node.
   * `View` > `Command Palette` > `Remote-SSH: Connect to Host...` > `+ Add New SSH Host` > `ssh myuser@dlcgpuxyz`
5. Once you're logged into the node, you can navigate to the directory of your repo in the Explorer (Ctrl+Shift+E / Cmd+Shift+E / `View` > `Explorer`)

You can now run and debug your code using the same workflows as when the code resides (and runs) on your local machine. Make sure you select your Python Interpreter (`Ctrl + Shift + P` > `Python: Select Interpreter`) from your conda environment before you run your code.

# Running Jupyter Notebook on a GPU on the cluster
You can run a Jupyter notebook on a GPU on the cluster and access it from the browser on your local machine using SSH port forwarding. Here's how you can do this (the instructions are taken from [this tutorial](https://alexanderlabwhoi.github.io/post/2019-03-08_jpn_slurm/)):

1. Run an interactive session as shown in the [section above](#running-an-interactive-session).
2. Once you are logged into a node, start your jupyter notebook.

   ```myuser@dlcgpu50:/path/to/my/directory$ jupyter notebook --no-browser --port=9001```

   Here, `myuser` is logged into node `dlcgpu50`, and starts a jupyter notebook which listens in on port 9001.
3. Create an SSH tunnel to the node that runs your Jupyter notebook.

   Open a new terminal on your local machine and:

   ```ssh -t -t myuser@kisxxx.xx.xx.xx -L 9001:localhost:9001 ssh dlcgpu50 -L 9001:localhost:9001```
4. Once the tunnel is established, you can open `localhost:9001` on your local machine to work on the Jupyter noteboook as if it is running on your machine, while still accessing the code and data that is available on the cluster.

## Making it easier
Steps 2 and 3 can be made easier by adding the following entries in your `.bash_profile` on the remote and the local machines.

1. On the remote machine (i.e., the cluster):
```
function jpt(){
    jupyter notebook --no-browser --port=$1
}
```

2. On your local machine:
```
function sshtojptnode(){
   # Forwards port $1 from node $2 into port $1 on the local machine and listens to it
   ssh -t -t youruser@kisxxxx.xx.xx.xxx -L $1:localhost:$1 ssh $2 -L $1:localhost:$1
}
```

Once these are ready, remotely running Jupyter notebooks is easy!

1. On the remote machine
```
myuser@dlcgpu13:/path/to/your/directory$ jpt 9001
```

2. On your local machine
```
sshtojptnode 9001 dlcgpu13
```

3. Now, simply fire up `localhost:9001` on your browser on your local machine.


# Cluster usage policy

* **Disk storage**: Always use [workspaces](https://kb.hlrs.de/platforms/index.php/Workspace_mechanism) for storing experiment data and for any I/O during a program run. This keeps the load on the login node low and allows faster I/O. Check `man ws_list`, `man ws_allocate` and `man ws_extend`.
* **Shared workspace**: When working on a collaborative project. Creating and executing [this script](https://gist.github.com/Neeratyoy/4cdf58f770164dfeea8be0e8d47fb6a7) allows read-write for other users.
* **Resource allocation request** (SBATCH parameter advice): It is important to know how much resources a job is requesting conditioned on the resource availability,
  * CPUs and GPUs requested: Check `sinfo` to see available resources. Resources requested should leave some resources free. If filling up a partition, the cluster [communication channel](https://im.tnt.uni-hannover.de/automl/channels/gpu-lovers) should be notified accordingly.
    * Each GPU requests 8 CPUs overriding the `--cpus-per-task` or `-c` flags
  * Memory requirement: Specifying `--mem` explicitly affects the resources requested in reality
    * A node has multiple CPU cores (~20) and the node RAM is split among these cores (~6 GB per CPU)
    * Requesting more than 6GB could actually request more than 1 CPU overriding the `--cpus-per-task` or `-c` flags
    * A good practice is to use `srun` or a test `sbatch` job to test actual memory requirements
  * *Alternatively*: to keep estimates easier, `-c` can be used directly to request resources (for e.g. `-c 2` == `--mem 12G` and `-c 4` == `--mem 24G`)
  * Time limits: Note the default timelimit of a job on a partition by `sinfo` when not specifying explicitly
    * The scheduler priority assigned to a job is often inversely proportional to the timilimit specified
* **Array jobs**: To prevent filling out a partition and leave resources for other users, using an [arrayjob](https://slurm.schedmd.com/job_array.html) is a must for multiple job deployments
  * Using `%n` and the above calculations of resource utilization, the near exact estimate of job runtime can be made
  * Post deployment, the resource request of a job can be updated
    * Update number of jobs in array job: `scontrol update ArrayTaskThrottle=[new n] JobId=[XXX]`
    * Jobs can be moved to an emptier partition: `scontrol update partition=[new partition] JobId=[XXX]`
* For any confusion regarding cluster usage and behaviour:
  * First search on the internet (or chatGPT of course)
  * Ask in the [Mattermost channel](https://im.tnt.uni-hannover.de/automl/channels/gpu-lovers)
  * Raise a ticket (if permissions exist) [here](https://osticket.informatik.uni-freiburg.de/tickets.php)

