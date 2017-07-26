# Using Nexus with Python

## Downloading packages from Nexus

[Pip](https://pip.pypa.io/en/stable/) can be configured to download packages from Nexus instead of PyPI. Most of the time this speeds up build processes (by caching commonly used packages) and helps ensuring reproducible builds.

To make `pip` use Nexus to download packages it is necessary to edit `pip`'s configuration file. This can be done on a per-user, per-_virtualenv_ or system-wide basis. For the remainder of this section we will assume a _per-user_ configuration, which provides a good balance and should be enough to cover most use cases. To use a different configuration style please refer to [`pip`'s documentation](https://pip.pypa.io/en/stable/user_guide/#config-file).

The per-user configuration file is located in different places on different operating systems:

* On macOS it lives in `${HOME}/Library/Application Support/pip/pip.conf`.
* On Linux it lives is `${HOME}/.config/pip/pip.conf`.
* On Windows it lives in `%APPDATA%\pip\pip.ini`.

Edit this file so that it reads

```ini
[global]
index = https://nexus.example.com/repository/pypi-all/pypi
index-url = https://nexus.example.com/repository/pypi-all/simple
```

This will make `pip` search for and install packages from the `pypi-all` group configured in the Nexus instance at `https://nexus.example.com`. You can check your configuration is working by issuing simple `pip` commands on verbose mode:

```text
$ pip2 search -vvv polyline
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

There may be some scenarios in which the Nexus instance is deployed behind a proxy which requires authentication. In these scenarios, the only way to preemptively convey authentication information is by specifying the credentials in the URL:

```ini
[global]
index = https://username:password@nexus.example.com/repository/pypi-all/pypi
index-url = https://username:password@nexus.example.com/repository/pypi-all/simple
```

From this moment on, it is of course recommended  to keep your `pip.conf` as safe from prying eyes as possible. On Unix systems, and if using the per-user configuration method we're assuming, you should `chmod 600` your configuration file. Also, make sure you use `pip`'s verbose mode with caution, as your credentials may end up in `stdout`/`stderr` or in some log file.

## Uploading packages to Nexus

Unlike in other languages and in other tools like Maven or Docker, package uploading in Python is handled by a different tool. Currently, [`twine`](https://github.com/pypa/twine) is the [recommended tool](https://github.com/pypa/twine#why-should-i-use-this). Uploading assumes, of course, that you have a pre-configured package. We recommend that you refer to [Packaging and Distributing Projects](https://packaging.python.org/tutorials/distributing-packages/#initial-files) for detailed steps on how your project structure should look like.

To upload packages to Nexus, include the following in your `${HOME}/.pypirc` (creating it if necessary):

```ini
[distutils]
index-servers =
   nexus

[nexus]
repository = https://nexus.example.com/repository/pypi-hosted/
username = your-username
password = your-pasword
```

Then, prepare your package for binary distribution:

```text
$ python setup.py sdist bdist_wheel
running sdist
running egg_info
(...)
Copying myproject.egg-info to build/bdist.macosx-10.12-x86_64/wheel/myproject-0.0.1.dev0-py2.7.egg-info
running install_scripts
creating build/bdist.macosx-10.12-x86_64/wheel/myproject-0.0.1.dev0.dist-info/WHEEL
```

This will generate a `dist/` directory in your project's root containing all the necessary artifacts for uploading. Upload them to Nexus using `twine`:

```text
$ twine upload dist/* -r nexus
Uploading distributions to https://nexus.example.com/repository/pypi-hosted/
Uploading myproject-0.0.1.dev0-py2-none-any.whl
Uploading myproject-0.0.1.dev0.tar.gz
```

Once your package is uploaded, it can be downloaded from other machines using `pip` and the abovementioned configuration.
