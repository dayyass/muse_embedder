[![tests](https://github.com/dayyass/muse-as-service/actions/workflows/tests.yml/badge.svg)](https://github.com/dayyass/muse-as-service/actions/workflows/tests.yml)
[![linter](https://github.com/dayyass/muse-as-service/actions/workflows/linter.yml/badge.svg)](https://github.com/dayyass/muse-as-service/actions/workflows/linter.yml)
[![codecov](https://codecov.io/gh/dayyass/muse-as-service/branch/main/graph/badge.svg?token=RRSTQY2R2Y)](https://codecov.io/gh/dayyass/muse-as-service)

[![python 3.6](https://img.shields.io/badge/python-3.6-blue.svg)](https://github.com/dayyass/muse-as-service#requirements)
[![release (latest by date)](https://img.shields.io/github/v/release/dayyass/muse-as-service)](https://github.com/dayyass/muse-as-service/releases/latest)
[![license](https://img.shields.io/github/license/dayyass/muse-as-service?color=blue)](https://github.com/dayyass/muse-as-service/blob/main/LICENSE)

[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-black)](https://github.com/dayyass/muse-as-service/blob/main/.pre-commit-config.yaml)
[![code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

[![pypi version](https://img.shields.io/pypi/v/muse-as-service)](https://pypi.org/project/muse-as-service)
[![pypi downloads](https://img.shields.io/pypi/dm/muse-as-service)](https://pypi.org/project/muse-as-service)

My public talk about this project at Sberloga:<br>
[**Web-service for Sentence Embeddings**](https://youtu.be/ZayiaA84oXg)

### What is MUSE?
**MUSE** stands for Multilingual Universal Sentence Encoder - multilingual extension (supports [16 languages](https://github.com/dayyass/muse-as-service#muse-supported-languages)) of Universal Sentence Encoder (USE).<br>
MUSE model encodes sentences into embedding vectors of fixed size.

- MUSE [paper](https://arxiv.org/abs/1907.04307)
- USE [paper](https://arxiv.org/abs/1803.11175)

### What is MUSE as Service?
MUSE as Service is the **REST API** for sentence tokenization and embedding using MUSE model from [TensorFlow Hub](https://tfhub.dev/google/universal-sentence-encoder-multilingual/3).

It is written using **Flask** and **Gunicorn**.

### Why I need it?
MUSE model from [TensorFlow Hub](https://tfhub.dev/google/universal-sentence-encoder-multilingual/3) requires next packages to be installed:
- tensorflow
- tensorflow-hub
- tensorflow-text

These packages take up more than **1GB** of memory. The model itself takes up **280MB** of memory.

For efficient memory usage when working with MUSE model on several projects (several virtual environments) or/and with teammates (several model copies on different computers) it is better to deploy one instance of the model in one virtual environment where all teammates have access to.

This is what **MUSE as Service** is made for! ❤️

### Requirements
Python >= 3.6

### Installation
To install MUSE as Service run:
```shell script
# clone repo (https/ssh)
git clone https://github.com/dayyass/muse-as-service.git
# git clone git@github.com:dayyass/muse-as-service.git

# install dependencies (preferable in venv)
cd muse-as-service
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip && pip install -r requirements.txt
```

Before using the service you need to:
- download MUSE model executing the following command:<br>
`
python models/download_muse.py
`

### Launch the Service
To build a **docker image** with a service parametrized with [gunicorn.conf.py](https://github.com/dayyass/muse-as-service/blob/main/gunicorn.conf.py) file run:
```shell script
docker build -t muse_as_service .
```
**NOTE**: instead of building a docker image, you can pull it from [Docker Hub](https://hub.docker.com/r/dayyass/muse_as_service).

To launch the service (either locally or on a server) use a **docker container**:
```shell script
docker run -d -p {host_port}:{container_port} --name muse_as_service muse_as_service
```
**NOTE**: `container_port` should be equal to `port` in [gunicorn.conf.py](https://github.com/dayyass/muse-as-service/blob/main/gunicorn.conf.py) file.

You can also launch the service without docker, but it is preferable to launch the service inside the docker container:
- **Gunicorn**: `gunicorn --config gunicorn.conf.py app:app` (parametrized with [gunicorn.conf.py](https://github.com/dayyass/muse-as-service/blob/main/gunicorn.conf.py) file)
- **Flask**: `python app.py --host {host} --port {port}` (default `host 0.0.0.0` and `port 5000`)

It is also possible to launch the service using [**systemd**](https://en.wikipedia.org/wiki/Systemd).

#### GPU support
MUSE as Service supports **GPU** inference. To launch the service with GPU support you need:
- install [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- use `CUDA_VISIBLE_DEVICES` environment variable to specify GPU device if needed (e.g. `export CUDA_VISIBLE_DEVICES=0`)
- launch the service with `docker run` command above (after `docker build`) with `--gpus all` parameter

**NOTE**: since **TensorFlow2.0** `tensorflow` and `tensorflow-gpu` packages are merged.

**NOTE**: depending on **CUDA** version installed you may need different `tensorflow` versions (default version `tensorflow==2.3.0` supports `CUDA 10.1`). See [table](https://www.tensorflow.org/install/source#gpu) with TF/CUDA compatibility to choose the right one and `pip install` it.

### Usage
Since the service is usually running on server, it is important to restrict access to the service.

For this reason, MUSE as Service uses **token-based authorization** with [JWT](https://jwt.io) for users in sqlite database [app.db](https://github.com/dayyass/muse-as-service/tree/main/src/muse_as_service/database/app.db).

Initially database has only one user with:
- **username**: "admin"
- **password**: "admin"

To add new user with `username` and `password` run:
```shell script
python src/muse_as_service/database/add_user.py --username {username} --password {password}
```
**NOTE**: no passwords are stored in the database, only their hashes.

To remove the user with `username` run:
```shell script
python src/muse_as_service/database/remove_user.py --username {username}
```

MUSE as Service has the following endpoints:
<pre>
- /login         - POST request with `username` and `password` to get tokens (access and refresh)
- /logout        - POST request to remove tokens (access and refresh)
- /token/refresh - POST request to refresh access token (refresh token required)
- /tokenize      - GET request for `sentence` tokenization (access token required)
- /embed         - GET request for `sentence` embedding (access token required)
</pre>

You can use python **requests** package to work with HTTP requests:
```python3
import numpy as np
import requests

# params
ip = "localhost"
port = 5000

sentences = ["This is sentence example.", "This is yet another sentence example."]

# start session
session = requests.Session()

# login
response = session.post(
    url=f"http://{ip}:{port}/login",
    json={"username": "admin", "password": "admin"},
)

# tokenizer
response = session.get(
    url=f"http://{ip}:{port}/tokenize",
    params={"sentence": sentences},
)
tokenized_sentence = response.json()["tokens"]

# embedder
response = session.get(
    url=f"http://{ip}:{port}/embed",
    params={"sentence": sentences},
)
embedding = np.array(response.json()["embedding"])

# logout
response = session.post(
    url=f"http://{ip}:{port}/logout",
)

# close session
session.close()

# results
print(tokenized_sentence)  # [
# ["▁This", "▁is", "▁sentence", "▁example", "."],
# ["▁This", "▁is", "▁yet", "▁another", "▁sentence", "▁example", "."]
# ]
print(embedding.shape)  # (2, 512)
```

However it is better to use built-in client **MUSEClient** for sentence tokenization and embedding, that wraps the functionality of the python **requests** package and provides user with a simpler interface.

To install the built-in client run:<br>
`
pip install muse-as-service
`

Instead of using endpoints, listed above, directly, **MUSEClient** provides the following methods to work with:
<pre>
- login    - method to login with `username` and `password`
- logout   - method to logout (login required)
- tokenize - method for `sentence` tokenization (login required)
- embed    - method for `sentence` embedding (login required)
</pre>

Usage example:
```python3
from muse_as_service import MUSEClient

# params
ip = "localhost"
port = 5000

sentences = ["This is sentence example.", "This is yet another sentence example."]

# init client
client = MUSEClient(ip=ip, port=port)

# login
client.login(username="admin", password="admin")

# tokenizer
tokenized_sentence = client.tokenize(sentences)

# embedder
embedding = client.embed(sentences)

# logout
client.logout()

# results
print(tokenized_sentence)  # [
# ["▁This", "▁is", "▁sentence", "▁example", "."],
# ["▁This", "▁is", "▁yet", "▁another", "▁sentence", "▁example", "."]
# ]
print(embedding.shape)  # (2, 512)
```

### Tests
To use [**pre-commit**](https://pre-commit.com) hooks run:<br>
`
pre-commit install
`

Before running tests and code coverage, you need to:
- run [app.py](https://github.com/dayyass/muse-as-service/blob/main/app.py) in background:<br>
`
python app.py &
`

To launch [**tests**](https://github.com/dayyass/muse-as-service/tree/main/tests) run:<br>
`
python -m unittest discover
`

To measure [**code coverage**](https://coverage.readthedocs.io) run:<br>
`
coverage run -m unittest discover && coverage report -m
`

**NOTE**: since we launched Flask application in background, we need to stop it after running tests and code coverage with the following command:
```shell script
kill $(ps aux | grep '[a]pp.py' | awk '{print $2}')
```

### MUSE supported languages
MUSE model supports next languages:
- Arabic
- Chinese-simplified
- Chinese-traditional
- Dutch
- English
- French
- German
- Italian
- Japanese
- Korean
- Polish
- Portuguese
- Russian
- Spanish
- Thai
- Turkish

### Citation
If you use **muse-as-service** in a scientific publication, we would appreciate references to the following BibTex entry:
```bibtex
@misc{dayyass2021muse,
    author       = {El-Ayyass, Dani},
    title        = {Multilingual Universal Sentence Encoder REST API},
    howpublished = {\url{https://github.com/dayyass/muse-as-service}},
    year         = {2021}
}
```
