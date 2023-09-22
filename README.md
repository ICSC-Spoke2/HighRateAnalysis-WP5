This repo is about serving the ASTRI ml deep learning classification software for 
performing inference on batches of stereo images, chosen by the user via a web interface
by selecting the energy bin.

## How the software is organized

This software has two main components, a frontend and a backend, each wrapped in its own docker. 
The two dockers mount the same `data` dir as a volume (need to overcome this: SEE LATER): data are only read, nothing is written. The data are png images - 56x56 the dimensions - representing stereo Cherenkov events as seen by the ASTRI mini-array, separated in two subdirs, `gamma` and `proton`, under an `inference` dir. In `data` there is also the CNN model, in `h5` format, and a sample of predictions to be used for frontend testing (in `notebooks` for example).

### Frontend

This is realized with the [`streamlit`](https://streamlit.io/) package.
It reads a configuration file (mounted as a volume in docker to avoid rebuilding because of a different config parameter.. ARE THERE BETTER PRACTICES?) where the gamma and proton dir together with the endpoint address are given.

```yaml
gamma_dir: "/path/to/gamma/"
proton_dir: "/path/to/proton/"
endpoint: "http://[ADDRESS]:8080/predict/"
```

There are three sliders the user can interact with, to select:

- the energy range (in buckets) of the events to predict
- the number of events to predict, ranging from 100 to 2000 in order to ensure a reasonable UX
- the number of sample images (up to 10) to display after the predictions

After pressing the _Predict!_ button, the frontend waits for the response, which is a json object with the classification probability for each of the selected event: with these data, the frontend build 4 plots for as many performance metrics, namely:

- a gamma/hadron separation plot in log scale
- a confusion matrix with threshold set at 90% probability
- a ROC curve with AUC score
- a precision/recall curve.

### Backend

The backend reads a config file (mounted in the docker the same way it is done for the frontend) with the following parameters

```yaml
gamma_dir: "/path/to/gamma/"
proton_dir: "/path/to/proton/"
model_file: "/path/to/model.h5"
database_file: "metadata.db"
img_w: 56
img_h: 56
port: 8080
```

It is realized in `python`, with the `fastapi` package for the enpoint building, served via the `uvicorn` package.

The `__fetch_and_inference` function is the heart of the backend, selecting the input data for the 
`predictor` object, then packing the predictions into two dictionaries, one for each particle type.

Here, a number of images with the energy range passed through the payload are selected from the metadata database, and then the inference stage is run on the images batch, loading a `tensorflow` model in `h5` format.
The two classes are handled separately (but in the same endpoint, since performance plots are meaningful with both of them predicted), and there's room for improving running them asyncronously.

## How to instance

The deployment is managed through a `docker-compose` file, and a proper `Makefile` to ease the typing.

Do

```sh
make build [frontend/backend]
```

to build the containers,

```sh
make up [frontend/backend]
```

to run them, 

```sh
make restart
```

to build and run both of them, and `make down` to shut them.

Docker images are stored on a proprietary (INAF) registry, and each component has its own script
`update_docker_registry.sh` to build and push a new image. 
The `docker-compose` file is really simple:

```yaml
services:
  frontend:
    build: frontend
    image: [REGISTRY_ADDRESS]:[PORT]/ml/inference_app/frontend
    ports:
      - 8501:8501
    # depends_on:
    #   - backend
    volumes:
        - ./data:/data
        - ${PWD}/config_frontend.yaml:/app/config_frontend.yaml
  backend:
    build: backend
    image: [REGISTRY_ADDRESS]:[PORT]/ml/inference_app/backend
    ports:
      - 8080:8080
    volumes:
      - ./data:/data
      - ${PWD}/config_backend.yaml:/app/config_backend.yaml
```

It can be seen that the docker image is fetched from the registry.
The `depends_on` clause is left here commented out, to highlight the application could run on a single
machine, as well as separating the two components (most common case, since the frontend does not benefit from a discrete GPU to run).

## Room for improvement

The application can be generalized at many levels: let's start with the performances.

### Optimization

Gammas and protons can undergo the inference stage asyncronously: both the results have to be waited for the API to return to the frontend, but they can go their way during the calcolous stage.

### Data: docker volumes? Streaming services?

As it can be seen in the `docker-compose`, the same `data` volume is mounted in both the containers, with different purposes: in the `backend`, files are needed as input for the `predictor`, which operates on a GPU and on batches of data. In the `frontend`, data files are just needed to be rendered on screen, so they could probably be fetched from an external service ([minio](https://min.io/)?).
There's the chance to use such a service for the inference stage as well: this should be investigated.

### Generalization

This kind of app is a (quite common nowadays) inference server.
It could be generalized by letting an user upload its own model and then the input file to predict over, customizing the dashboard accordingly.# HighRateAnalysis-WP5
