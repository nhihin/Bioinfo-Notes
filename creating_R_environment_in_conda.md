# Creating R environment in conda

## Setting up the environment

These instructions assume that Anaconda is already installed (or can be loaded using `module load`, e.g. `module load Anaconda3/5.1.0` on Phoenix HPC). 

Create a new conda environment for the R install:

```
conda create --name R_env
```

Activate the environment using `source activate R_env` and install the following in this order:

```
conda install r-essentials r-base
conda install -c conda-forge r-devtools
conda install -c conda-forge r-biocmanager
```

Going in this order ensures that the version of R is 3.6+, which is required for some Bioconductor packages.

Type `R` to start R. 

Because the BiocManager may be out of date, use the command `BiocManager::valid()` and copy and paste the command it outputs in order to get the packages to be valid. This may take a while. 

Desired packages can now be installed using `install.packages()` within R or using conda, which is the safer option. Note, if packages installed using `install_github` gives the following error:

```
> install_github("satijalab/Seurat", ref="develop")
Using GitHub PAT from envvar GITHUB_PAT
Downloading GitHub repo StatsWithR/statsr@master
from URL
Installation failed: Bad credentials (401)
```

This can be resolved by unsetting the envvar GITHUB_PAT by using this command: `Sys.unsetenv("GITHUB_PAT")`, see [ref](https://github.com/r-lib/devtools/issues/1566). 







