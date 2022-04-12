# Using Jupyter Notebooks on HPC

## Setup - Create a conda environment

- First create a conda environment containing all the Python libraries that are later to be used in the Jupyter notebook. 

e.g. `conda create --name SCVELO_env python=3.6 scvelo`

- Activate the environment (`conda activate SCVELO_env`) and then install Jupyter Notebook using `conda install -c conda-forge notebook`. 

## Adding Jupyter as a SLURM job on HPC

- **Note**: The following is adapted to SAHMRI HPC, which is why `edp-prd-lin-hpc01` is used for the ssh part below. If using on Phoenix, this needs to be changed accordingly. Adapted from https://docs.ycrc.yale.edu/clusters-at-yale/guides/jupyter/

- Make a file called `start_jupyter.sh` that contains the following:

```
#!/bin/bash

#SBATCH --job-name=SCVelo_Notebook
#SBATCH --mail-user=nhi.hin@sahmri.com
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --output=velocyto1.log

# Resources allocation request parameters

#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=32
#SBATCH --mem 128000M
#SBATCH --time=48:00:00          # Run time in hh:mm:s



# get tunneling info
XDG_RUNTIME_DIR=""
port=$(shuf -i8000-9999 -n1)
node=$(hostname -s)
user=$(whoami)

# print tunneling instructions jupyter-log
echo -e "\
Use this command in your terminal:\
ssh -N -L ${port}:${node}:${port} ${user}@edp-prd-lin-hpc01\

Use a Browser on your local machine to go to:\
localhost:${port}  (prefix w/ https:// if using password)
"

# Start notebook

source activate SCVELO_env
export XDG_RUNTIME_DIR=""
jupyter notebook --no-browser --port=${port} --ip=${node}
source deactivate
```

- After submitting this job to the SLURM queue, do `cat velocyto1.log` (or whatever the log file is named in the script above), and it should look something like this:

```
Use this command in your terminal:ssh -N -L 8320:edp-prd-lin-hpc01:8320 nhi.hin@edp-prd-lin-hpc01
Use a Browser on your local machine to go to:localhost:8320  (prefix w/ https:// if using password)

[I 16:40:08.935 NotebookApp] Serving notebooks from local directory: /homes/nhi.hin/WORKING/velocyto
[I 16:40:08.935 NotebookApp] Jupyter Notebook 6.2.0 is running at:
[I 16:40:08.936 NotebookApp] http://edp-prd-lin-hpc01:8320/?token=f2c78b1c2878346ab69d344f323a1cdc3eff9868906b33d4
[I 16:40:08.936 NotebookApp]  or http://127.0.0.1:8320/?token=f2c78b1c2878346ab69d344f323a1cdc3eff9868906b33d4
[I 16:40:08.936 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 16:40:08.993 NotebookApp]

    To access the notebook, open this file in a browser:
        file:///homes/nhi.hin/.local/share/jupyter/runtime/nbserver-15298-open.html
    Or copy and paste one of these URLs:
        http://edp-prd-lin-hpc01:8320/?token=f2c78b1c2878346ab69d344f323a1cdc3eff9868906b33d4
     or http://127.0.0.1:8320/?token=f2c78b1c2878346ab69d344f323a1cdc3eff9868906b33d4
```

- Open a new Terminal tab which is NOT logged into the HPC, and and copy and paste the command from that log file, in my case, mine is `ssh -N -L 8320:edp-prd-lin-hpc01:8320 nhi.hin@edp-prd-lin-hpc01`. 

- Open a web browser and start the Jupyter Notebook using the URL stated in the log file, in my case, I would use `http://127.0.0.1:8320/?token=f2c78b1c2878346ab69d344f323a1cdc3eff9868906b33d4`.

- To exit the notebook, cancel the SLURM job. 
