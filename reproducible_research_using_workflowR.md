# Reproducible Research and R Projects using WorkflowR

## Step 0 - Initialisation (optional - for increased reproducibility)

**NOTE**: I **highly** recommend skipping this step for most projects nowadays as re-installing everything takes ages. Will use for projects where there is an important need to have everything be 100% reproducible. 

- Create a new project as normal in RStudio (without git initialisation, as WorkflowR will create one later). 
- Before writing any code, load the Packrat R package `library(packrat)` and initialise Packrat using `packrat::init("~/pathTo/PROJECTNAME")`. Packrat is a library manager for R and RStudio that isolates the packages used to a particular project. This ensures that the exact packages and their versions that were used in a particular project can be carried in a modular way with the project, making it easy to reproduce. More information about Packrat can be found in their [documentation](https://rstudio.github.io/packrat/walkthrough.html). 
- After installing new packages with Packrat (using `install_packages()`), use `packrat::snapshot()` to save the changes. 

## Step 1 - Setting up WorkflowR 

We will use WorkflowR to manage and organise the directories and files for the project. 

1. Create a new project in RStudio (File > New Project). Do not initialise git since WorkflowR will do that later. 
2. Install [workflowR](https://jdblischak.github.io/workflowr/articles/wflow-01-getting-started.html) and load the package using `library(workflowr)`. 
3. Run the following in the Console to set up WorkflowR:

```{r}
wflow_start("~/pathTo/PROJECTNAME", existing = TRUE, overwrite = TRUE)
```

4. Run `wflow_build()` in the Console. 
5. Build and commit the website using `wflow_publish()`. 

**Note**: The status of files can be checked using `wflow_status()`.

6. To deploy the website, run the following in Console: `wflow_use_github("nhihin")` (replace with GitHub username) and select option 1 to automatically create the repository. **NOTE**: When you have to enter your password for this step, instead of your GitHub password, create a Personal Access Token [here](https://github.com/settings/tokens) and enter the value of that token instead! This is because GitHub is deprecating the use of passwords directly by 3rd party programs (including RStudio), see [here](https://developer.github.com/changes/2020-02-14-deprecating-password-auth/). 
7. The Git tab on RStudio can now be used as normal to commit, push, and pull files. If it isn't showing up, usually restarting RStudio fixes this.

**Note**: If the GitHub repository needs to be private (e.g. sensitive/client data), this needs to be changed immediately after step 6 on the GitHub website, see [here](https://docs.github.com/en/github/administering-a-repository/setting-repository-visibility#:~:text=On%20GitHub%2C%20navigate%20to%20the,Select%20a%20visibility.). 

### Making a new analysis file

- Use `wflow_open("analysis/your-file-name.Rmd")` to make a new analysis RMarkdown. 
- Files must be committed using `wflow_build("analysis/your-file-name.Rmd")` and pushed using `wflow_publish("analysis/your-file-name.Rmd")`.
- Globbing is useful for committing and pushing, for example, `wflow_build("analysis/*.Rmd")` will build all .Rmd files in the analysis directory. 

### Editing the navigation bar

- This is important to do early on, as the navigation bar defined will be used across every analysis file created. This saves time if defined early on. 

1. Open the file in `analysis/_site.yml`. 
2. Under `left:` is where you can define links in the navigation bar. `Text` is the name of the link while `href` is the URL of the link. This is a relative url pointing to the `analysis` directory so no need to put in the full path.
3. Typically one link for one analysis file for each specific step of the analysis. The `menu:` thing is used to define a drop-down menu, which is helpful for organising many analysis pages which are related to each other. 

Example:

```
name: Project Name
output_dir: ../docs
navbar:
  title: Project Name
  left:
  - text: Home
    href: index.html
  - text: About
    href: about.html
  - text: Setup
    href: setup.html
    menu:
      - text: "Pre-processing"
      - text: "cellranger pipeline"
        href: cellranger.html
      - text: "Initial analysis"
      - text: "Merge datasets with Seurat"
        href: import-data.html
      - text: "Aggregate with cellranger"
        href: import-merged-data.html
  - text: License
    href: license.html
  right:
  - icon: fa-github
    text: Source code
    href: https://github.com/nhihin/Heart-scRNAseq
output:
  workflowr::wflow_html:
    toc: yes
    toc_float: yes
    theme: cosmo
    highlight: textmate
```

## Publishing the website on GitHub pages

- Once all RMarkdown files have been published as .html files (and pushed to GitHub using `wflow_publish`), go to `Settings > GitHub Pages` and choose “master branch docs/ folder” as the Source. See screenshot below.

![](https://i.imgur.com/T6PiQSA.png)

- This will publish the files in `docs` under `nhihin.github.io/myproject` (or the URL listed on GitHub Pages section of the Settings page). See https://jdblischak.github.io/workflowr/articles/wflow-01-getting-started.html#deploy-the-website for more details.

- Gitlab is an alternative way to deploying the website and having it hosted online. Instructions here: https://jdblischak.github.io/workflowr/articles/wflow-06-gitlab.html

