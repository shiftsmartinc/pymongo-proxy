# pymongo-proxy

Shiftsmart-owned fork of the [official MongoDB Python driver
(PyMongo)](https://github.com/mongodb/mongo-python-driver) with HTTP/SOCKS proxy
support via [python-socks](https://github.com/romis2012/python-socks).

**Baseline:** PyMongo **4.19.0**

DNS for the MongoDB host is still resolved by PyMongo. When `MONGO_PROXY` is set,
the TCP connect to Mongo goes through the proxy (python-socks). When unset,
behavior matches upstream PyMongo.

This exists so Vertex AI pipeline workers — which cannot open a direct TCP path
to MongoDB from the VPC — can set `MONGO_PROXY` from Secret Manager
(`prod_vertex_mongo_proxy_uri`) and keep using stock `MongoClient(uri)`. The
network proxy itself is separate infra; this repo is only the client driver.

Behavior and env contract match the earlier
[benjamintenny/mongo-python-driver-proxy](https://github.com/benjamintenny/mongo-python-driver-proxy)
fork so `pipelines-core[mongo]` can switch the git source without other code
changes.

## Install

Uninstall stock pymongo first if present:

```bash
pip uninstall pymongo
```

From git:

```bash
pip install git+https://github.com/shiftsmartinc/pymongo-proxy.git
```

Or from a local checkout:

```bash
pip install .
```

Pin a commit or tag in lockfiles when used from `pipelines-core[mongo]`.

## Usage

```python
import os
import pymongo

# Set socks5 proxy (Vertex pipelines load this from Secret Manager)
os.environ["MONGO_PROXY"] = "socks5://127.0.0.1:1080"
client = pymongo.MongoClient("your-mongodb-url")
db = client.test
print(db.name)
```

Socks5 with authentication:

```python
os.environ["MONGO_PROXY"] = "socks5://username:password@127.0.0.1:1080"
```

HTTP proxy with authentication:

```python
os.environ["MONGO_PROXY"] = "http://username:password@127.0.0.1:8080"
```

Socks4 proxy:

```python
os.environ["MONGO_PROXY"] = "socks4://127.0.0.1:1080"
```

## How to bump against upstream

1. Fetch the desired tag from `mongodb/mongo-python-driver`.
2. Check out / merge that tag into this repo.
3. Re-apply the `MONGO_PROXY` patches in `pymongo/pool_shared.py`
   (`_create_connection` / `_async_create_connection`) and keep
   `python-socks` in `requirements.txt`.
4. Confirm `pymongo/_version.py` matches the upstream baseline.
5. Run `uv lock` here, then retarget / relock consumers in `ml-models`.

## Upstream docs

See [README_ORIGINAL.md](README_ORIGINAL.md) for the upstream PyMongo README.
