## Creating a Shiny App

- Refer to [Shiny documentation](https://shiny.rstudio.com/tutorial/) for tutorials on how to create a Shiny app. 

## Deploying a Shiny App on shinyapps.io (Easy Method)

- **NOTE**: Deploying an app is much easier to shinyapps.io compared to the Docker + Google Cloud Run method that will be described later. I would recommend this route unless the memory limits on the free tier are inadequate. Unfortunately, the paid plans for shinyapps.io are quite expensive compared to other providers (e.g. AWS, Google Cloud Run) but I guess it might be worth it for the convenience in some cases. 

- Create an account on shinyapps.io. 
- Install the `rsconnect` package on R (`install.packages("rsconnect")`). 
- Authorise your computer to connect to your shinyapps.io account by copying the command that shinyapps.io gives you. It should look something like this:

```
rsconnect::setAccountInfo(name='nhin', token='ASDFASDFASDF', secret='ASDFASDFASDFASDF')
```

- Now when you are in your Shiny app project in RStudio, you can use the blue-coloured button next to the "Run App" button to deploy your Shiny app (see screenshot below), and follow the instructions in the prompt. 

![](https://i.imgur.com/kCs7cNv.png)

- Note that the free tier on shinyapps.io only allows for 1Gb RAM max, but sometimes the default option is not set to 1Gb. To enable this, log in to shinyapps.io in a web browser, go to `Applications > (Click on the name of your app) > Settings` and change the `Instance Size` to Large. 

![](https://i.imgur.com/5SnTEFZ.png)

## Deploying a Shiny App on Google Cloud Run

- To deploy a Shiny app on Google Cloud Run or any other online provider (e.g. AWS, Heroku etc), the app first needs to be deployable from a container, i.e. Docker. 

### Running Shiny App in Docker

- Refer to https://www.statworx.com/at/blog/how-to-dockerize-shinyapps/ for details of the method used. It is important to install `renv` and use it exactly as the tutorial states in order to save the R packages being used in the project, so that we can call upon that in the Dockerfile later.

- Next, open a text editor and create a file named `Dockerfile` containing the following information:

```
# get shiny serves plus tidyverse packages image
FROM rocker/shiny-verse:latest

# system libraries of general use
RUN apt-get update && apt-get install -y \
    sudo \
    pandoc \
    pandoc-citeproc \
    libcurl4-gnutls-dev \
    libcairo2-dev \
    libxt-dev \
    libssl-dev \
    libssh2-1-dev 

## update system libraries
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get clean

# copy all app files in current directory to the docker image
COPY . ./app
## renv.lock file
COPY renv.lock ./renv.lock

# install renv & restore packages
RUN Rscript -e 'install.packages("renv")'
RUN Rscript -e 'renv::consent(provided = TRUE)'
RUN Rscript -e 'renv::restore()'

# select port
EXPOSE 3838

# run app
CMD ["R", "-e", "shiny::runApp('/app', host = '0.0.0.0', port = 3838)"]
```

- The next step is to build the Docker image as follows:

```
docker build -t myapp . 
```

- Lastly, the container can be started by running the following:

```
docker run -d --rm -p 3838:3838 myapp
```

- By going to `localhost:3838` in a web browser, you should be able to see the Shiny app. 

### Setting up Google Cloud Run

- The first step is to [make an account on Google Cloud](https://cloud.google.com/) and make your first project. 

- Most of the following steps will be done on the command line using `gcloud`, which can be installed with the [steps here](https://cloud.google.com/sdk/docs/install) (ensure you are using the specific steps for your OS). Next, on a command line, log in to your Google Cloud account using:

```
gcloud auth login
```

- Next you will want to log onto your Google Cloud account on a web browser and get the name of your project. It looks like a string of words with some numbers on the end, e.g. `banana-chemist-34524`. Copy this and use it in the following command:

```
gcloud config set project banana-chemist-34524
```

- We now tag the Docker container containing the Shiny app (which should have already been built) using the following command. Note the following important points: 

   - `myapp` should be replaced with whatever you want to name your service (Shiny app). This name must be in lowercase and only contain letters, numbers or hyphen (-). Some valid names are e.g. `nhi`, `scrna-seq`. 

   - The string of numbers/letters in the command below corresponds to your Docker image ID, which can be found by running `docker image list` and copying the string of numbers/letters under `IMAGE ID`. 

```
docker image tag 0fcc82dfgddf1 gcr.io/banana-chemist-34524/myapp
```

- Run the following lines to set up the authentication properly. This ensures that your pre-built Docker container on your machine has the permissions to push it up onto Google Cloud's container registry. 

```
sudo usermod -a -G docker ${USER}
gcloud auth configure-docker
gcloud auth print-access-token | docker login -u oauth2accesstoken --password-stdin gcr.io
```

- Now we can finally push the Docker container up to Google Cloud's container registry.

```
docker push gcr.io/banana-chemist-34524/myapp
```

- The final step is the deployment step, which basically makes the app public online. This can either be done through command line (`gcloud`) or through the web console. I have only done it through the web console. Essentially, log onto the **Google Cloud** console and ensure you are in the **Google Cloud Run** area.

- Click **Create Service** and follow the prompts. Set the port to `3838` and leave everything else as default. 

