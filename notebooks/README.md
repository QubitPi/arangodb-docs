ArangoDB Interactive Tutorials
==============================

Here are tutorials and interactive notebooks for ArangoDB features. These notebooks allow us to learn about ArangoDB
concepts and features without the need to install or sign up for anything. Each notebook uses a free temporary tutorial
database that lets us focus on all the cool things ArangoDB has to offer.

Notebooks
---------

- ArangoDB

  - [AQL](./AQL.ipynb)
  - [Arango Search](./ArangoSearch.ipynb)

Setup
-----

### Getting Source Code

```console
git clone git@github.com:QubitPi/arangodb-docs.git
cd notebooks
```

### Creating an Isolated Environment

It is strongly recommended to work in an isolated environment. 

> [!IMPORTANT]
> 
> If this is the very first time for initializing the environment, please install `virtualenv` and create an isolated
> Python environment by
> 
> ```console
> python3 -m pip install --user -U virtualenv
> python3 -m virtualenv .venv
> ```

Activate environment defined by each `requirements.txt` with:

```console
source .venv/bin/activate
```

or, on Windows

```console
./venv\Scripts\activate
```

> [!TIP]
> 
> To deactivate this environment, use
> 
> ```console
> deactivate
> ```

Install dependencies by

```console
pip3 install -r requirements.txt
```

### Starting Jupyter Server

Now we can fire up Jupyter by typing the following command:

```console
python3 -m ipykernel install --user --name=python3 && jupyter notebook
```

The first half of the composite commands registers virtualenv to Jupyter and give it a name with `python3`

A Jupyter server is now running in our terminal, listening to port 8888. We can visit this server by opening our web
browser to http://localhost:8888/
