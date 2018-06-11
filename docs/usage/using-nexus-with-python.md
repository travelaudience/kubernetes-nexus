# Using Nexus with Python

Configuring [`pip`](https://pip.pypa.io) to download artifacts from Nexus instead
of Pypi will, most of the time, not only speed up build processes by
caching commonly used dependencies but also help ensuring reproducible builds,
since one only depends on their Nexus availability and not the public repositories.

`pip` can also be configured to upload artifacts to Nexus, enabling the management
of artifacts private to an organization.

## Downloading packages from Nexus

In order to enable `pip` to download packages from Nexus, it is necessary to
edit `pip` configuration file. This can be done on a per-user, per-_virtualenv_
or system-wide basis.
For the remainder of this section we will assume a _per-user_ configuration.
To use a different configuration, one should refer to
[`pip`'s documentation](https://pip.pypa.io/en/stable/user_guide/#config-file).

The per-user configuration file is located in different places on different OS'es:

* On Mac OS: `${HOME}/Library/Application Support/pip/pip.conf`.
* On Linux: `${HOME}/.config/pip/pip.conf`.
* On Windows: `%APPDATA%\pip\pip.ini`.

Edit the corresponding file as follows:

```ini
[global]
index = https://nexus.example.com/repository/pypi-all/pypi
index-url = https://nexus.example.com/repository/pypi-all/simple
no-cache-dir = false 
```

This will instruct `pip` to search for and install packages from the `pypi-all`
group, previously configured in Nexus.
One can check if their configuration is correct by running:

```shell
$ pip2 search -vvv polyline
```

One should see feedback like:

```
Starting new HTTPS connection (1): nexus.example.com
"POST /repository/pypi-all/pypi HTTP/1.1" 200 2742
cGPolyEncode (0.1.1)         - Google Maps Polyline encoding (C++ extension)
GPolyEncode (0.1.1)          - Google Maps Polyline encoding (pure Python)
gpolyline (0.0.3)            - Converts a series of latitude/longitude points to an encoded string for use with Google Maps
polyline (1.3.2)             - A Python implementation of Google's Encoded Polyline Algorithm Format.
pypolyline (0.1.11)          - Fast Google Polyline encoding and decoding using Rust FFI
SkyLinesPolyEncode (0.1.3)   - SkyLines Polyline encoding (C++ extension)
time_aware_polyline (0.1.2)  - Time aware encoded polyline for geospatial data
```

There may be some scenarios in which the Nexus is deployed behind a proxy which requires authentication. In these scenarios, the only way to preemptively convey authentication information is by specifying the credentials in the URL:

```ini
[global]
index = https://username:password@nexus.example.com/repository/pypi-all/pypi
index-url = https://username:password@nexus.example.com/repository/pypi-all/simple
no-cache-dir = false 
```

**Attention:** If GCP IAM authentication is enabled, [username and password
**are not** the GCP organization credentials](../admin/configuring-nexus-proxy.md/#usage).

From this moment on, it is of course recommended  to keep `pip.conf` as safe as possible.
On Unix systems one should `chmod 600` the configuration file. Also, one is to make sure
to use `pip`'s verbose mode with caution, as credentials may end up in `stdout`/`stderr`
or in some log file.

## Uploading packages to Nexus

Unlike other tools, such as Maven or Docker, package uploading in Python is handled
not by the `pip` tool, the tool we adopted to manage Python depdencies, but by a different tool.
Currently, [`twine`](https://github.com/pypa/twine) is the [recommended tool](https://github.com/pypa/twine#why-should-i-use-this).
We recommend that one refers to [Packaging and Distributing Projects](https://packaging.python.org/tutorials/distributing-packages/#initial-files) for detailed steps on how a project structure should look like.

To upload packages to Nexus, one must include the following in `${HOME}/.pypirc` (create it it if necessary):

```ini
[distutils]
index-servers =
   nexus

[nexus]
repository = https://nexus.example.com/repository/pypi-hosted/
username = the-username
password = the-pasword
```

**Attention:** If GCP IAM authentication is enabled, [username and password
**are not** the GCP organization credentials](../admin/configuring-nexus-proxy.md/#usage).

Then, prepare the package for binary distribution:

```shell
$ python setup.py sdist bdist_wheel
```

The above will generate a `dist/` directory in the project's root containing all
the necessary artifacts for uploading. One is to upload the same artifacts to
Nexus using `twine`:

```shell
$ twine upload dist/* -r nexus
```

Once the package is uploaded, it can be downloaded from other machines using `pip`
for as long as it's configured as instructed above.
