# Getting started

Computing is performed on Yale's [Bouchet](https://docs.ycrc.yale.edu/clusters/bouchet/) cluster. This page contains instructions on how to setup Julia on [bouchet](https://docs.ycrc.yale.edu/clusters/bouchet/). 

## Getting started with Bouchet

Make sure you have an account on Bouchet! You can request an account [here](https://research.computing.yale.edu/account-request).

Once an account is created you can log `ssh` into Bouchet with your net ID as the username:

```
ssh netid@bouchet.ycrc.yale.edu
```

??? note "Example Login"

    For example, my net ID is `ljg48` so I would type the following into the terminal:

    ```
    ssh ljg48@bouchet.ycrc.yale.edu
    ```

If this is your first time using Bouchet I strongly encourage you to look over Bouchet's [Getting Strated](https://docs.ycrc.yale.edu/clusters-at-yale/) pages. If you have any questions either reach out to YCRC or ask Elizabeth or Luke.

YCRC uses the Duo multi-factor authentication (MFA), the same that is used elsewhere at Yale. To get set up with Duo, follow these [instructions](https://cybersecurity.yale.edu/mfa).

Yale's clusters can only be accessed on the Yale network. Thus, you will need to connect to Yale's VPN before SSH'ing into Bouchet. See the [ITS webpage](https://yale.service-now.com/it?id=service_offering&sys_id=c4684dcd6fbb31007ee2abcf9f3ee4f2) for more details. YCRC has some recommendations for VPN software [here](https://docs.ycrc.yale.edu/clusters-at-yale/access/vpn/)

## Setting up Julia

1. SSH into Bouchet with `ssh netid@bouchet.ycrc.yale.edu`
2. Once connected, log into the `devel` partition `salloc -c4 -p devel`
3. Load miniconda `module load miniconda`
4. Create a new `conda` enviroment, useful if you use [OOD](https://docs.ycrc.yale.edu/clusters-at-yale/access/ood/) `conda create --name climaocean python jupyter jupyterlab`. Here I named it climaocean, but you can choose anything you like, and I'm adding python, jupyter, and jupyterlab
5. Make the environment a kernel so you can use it with notebooks with the following command: `ycrc_conda_env.sh update`
6. Activate your new environment: `conda activate climaocean`
7. Load Julia `module load Julia/1.11.4-linux-x86_64`
8. Start the Julia REPL: `julia`
9. Once in the REPL, type `]` to go into the `Pkg` environment
10. From here, type `add IJulia` to add the IJulia kernel. This is so you can use Julia with Jupyter
11. Now, this next part is **very important**, if you start an [OOD](https://docs.ycrc.yale.edu/clusters-at-yale/access/ood/) session make sure to include `Julia/1.11.4-linux-x86_64` under `additional modules` and **make sure to activate the environment you just created**.

## Julia Depot Path

By defult, `~/.julia/` is the Julia depot path, the directory where all packages, configurations, and other Julia-related files are stored. This is set by the `JULIA_DEPOT_PATH` envionrment variable. Since there is limited space in your home directory on Bouchet, it best to change the depot to something like scratch. 

```sh
DEPOT_PATH=/home/${USER}/project/JULIA_DEPOT
mkdir -p ${DEPOT_PATH}
export JULIA_DEPOT_PATH=${DEPOT_PATH}
```

??? note "Alternatively, use symlink"

    1. move `.julia` to scratch `mv /home/${USER}/.julia /home/${USER}/scratch/.julia` 
    2. symlink to your home direcotry `ln -s /home/${USER}/project/.julia /home/${USER}/.julia`

## Create a Julia environment

1. Navigate to where you want your environment to live. I suggest a folder in your project directory
2. Type `julia` to start to the REPL
3. Then type `]` to start the package manager
4. Type `activate .` to set the current working directory as the active environment. This creates a `Project.toml` and `Manifest.toml`, two files that include information about dependencies, versions, package names, UUIDs etc.
5. Add packages to the environment: `add ClimaOcean OceanBioME`
6. Once that is complete, press `delete` to go to the julia repl then type `exit()` to close out the REPL

!!! note

    When you add packages the the code is stored in the directory defined by `JULIA_DEPOT_PATH`. 
    On Bouchet this should be the scratch directory since it can get large. 
    The downside this directory is purged after 90 days. 


## Running a simple model

Now that everything is setup, let's go through the steps of setting of a simple simulation and submitting the run to the cluseter.


### Setup simulation

### Submit to cluster

Open the note below to show the submit.sh script used to submit a job to the cluster. Copy the code store it in your project directory. If you are new to slurm, I suggest looking at the [YCRC documentation](https://docs.ycrc.yale.edu/clusters-at-yale/job-scheduling/)

??? note "submission script"

    ```sh title="submit.sh"
    #!/bin/bash
    #SBATCH --job-name=clima
    #SBATCH --ntasks=1
    #SBATCH --time=3:00:00
    #SBATCH --account eisaman
    #SBATCH --nodes 1
    #SBATCH --mem 10G
    #SBATCH --partition gpu
    #SBATCH --gpus=a100:1

    module purge
    module load miniconda/24.9.2 
    module load Julia/1.11.4-linux-x86_64

    ###---------------------------------------------------------
    ### if true, this will instantiste the project
    ### set to true only if the project is not instantiated yet
    ### this will only need to be done once
    ###---------------------------------------------------------

    INSTANTIATE=false

    ###---------------------------------------------------------
    ### path to simulation you want to run
    ###---------------------------------------------------------

    SIMULATION=/home/${USER}/project/repos/oceananigans-coupled-global/simulations/global-simulation-dev.jl 

    ###---------------------------------------------------------
    ### this is path where where Project.toml is
    ### note: do not put Project.toml at end of the path
    ###---------------------------------------------------------

    PROJECT=/home/${USER}/project/repos/oceananigans-coupled-global/

    ###---------------------------------------------------------
    ### this is where all downloaded file will live
    ### will make scratch directory if does not already exist
    ###---------------------------------------------------------

    DEPOT_PATH=/home/${USER}/project/JULIA_DEPOT

    mkdir -p ${DEPOT_PATH}

    export JULIA_DEPOT_PATH=${DEPOT_PATH}

    ###-------------------------------------------
    ### this contains ECCO credentials
    ### your ~/.ecco-credentials 
    ### should contain only these two lines:
    ###
    ### export ECCO_USERNAME=your-username
    ### export ECCO_PASSWORD=your-password
    ###-------------------------------------------

    source /home/${USER}/.ecco-drive  

    ###-------------------------------------------
    ### instantiates packages
    ### should only have to run this once
    ###-------------------------------------------

    if ${INSTANTIATE}; then
        julia --project="${PROJECT}" -e "using Pkg; Pkg.instantiate()"
    fi

    wait

    ###-------------------------------------------
    ### runs the actual simulation
    ###-------------------------------------------

    julia --project=${PROJECT} ${SIMULATION}
    ```

`submit.sh` is a slurm script the tells the cluster the resources needed to run the job and what you want it to run. It is good practice to run jobs on a compute node and save unprocessed output to `scratch`. 

**Submit Job To Cluster**

```sh title="submits job to cluster"
sbatch submit.sh
```

While a job is running, output logs are saved to `slurm-jobID.out`. You can `tail` this file to checkup on the running simulation.

**Check Status of a Job**

```sh title="check status of just your jobs"
squeue --me
```

**Efficiency Report of Completed Job**

```sh title="show how effecient you used the resources when job completes"
seff jobID
```

!!! note

    The effeciency report can help you more effeciently use resources for future runs. Remember, Bouchet is a shared cluster. If you request nodes that are not being utilized then nobody else can use them until your job is complete. Be mindful of others.

