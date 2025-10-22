# docker-devpi

Docker images for both [`devpi-server`][1] and [`devpi-client`][2], with both
Debian and Alpine versions available.

[Devpi][3] allows you to host a local PyPi cache, along with the ability for you to
add your own packages that you do not want to upload publicly.

> :information_source: Docker tags use the same version numberings as the devpi
> packges on PyPi ([server][5] & [client][6]), but can be viewed on DockerHub
> as well: [server][7] & [client][8].


# Usage

This repository is split into two parts: the [server](#server) and the
[client](#client). You will of course need a working server before there is
any point in using the client.

## Server

###  Environment Variables
- `DEVPI_PASSWORD`: The password to set for the "root" user on init (**required**).

### Run it

The `DEVPI_PASSWORD` environment variable will set the root password on the
first startup of this image. Set it to something more secure than what I use
as an example here.

```bash
docker run -it -p 3141:3141 \
    -e DEVPI_PASSWORD=password \
    -v $(pwd)/data-server:/devpi/server \
    --name devpi-server loveyu/devpi-server:latest
```

The command will also host mount the data directory from the server to your
local folder, in order to persist data between restarts.

> The first time this container is started a full re-index of the upstream PyPi
> will commence, so the logs might be spammed with "Committing 2500 new
> documents to search index." for a while.

It is possible to customize the devpi instance with any of the available
[arguments][1] by appending them to the command displayed above:

```bash
docker run -it <...> loveyu/devpi-server:latest \
    --threads 4 \
    --debug
```


## Client

###  Environment Variables
- `DEVPI_URL`: The URL to the devpi instance to connect to (default: `http://localhost:3141/root/pypi`)
- `DEVPI_USER`: The username to login to the devpi instance with (default: `""`)
- `DEVPI_PASSWORD`: The password for the user when logging in (default: `""`)

### Run it

The `DEVPI_URL` environment variable is the only one that needs to be set, and
it will use the default unless you specify something else. The other two are
optional, and can be defined for convenience so you do not have to manually
login every time you start this container. Furthermore, if `DEVPI_USER` is set
but `DEVPI_PASSWORD` is empty you will be prompted for the password.

The following command expects that there is a functional devpi instance running
on the local computer, and that the "root" user has a password set. The host
mounted directory may be used to transfer files in and out of the container.

```bash
docker run -it --rm --network host \
    -e DEVPI_USER=root \
    -e DEVPI_PASSWORD=password \
    -v $(pwd)/data-client:/devpi/client \
    loveyu/devpi-client:latest
```

Important note here is that this container uses the "host" network, else the
`localhost` in the URL would just make so that the container tried to contact
itself and would thus not reach the container running the server program.


## Configure pip

For a quick test to see if the `devpi-server` actually works you can make a
one-off installation like this:

```bash
pip install -i http://localhost:3141/root/pypi simplejson
```

The server logs should move and the `pip` installation should be successful.

To then make this setting a bit more permanent you can edit `~/.pip/pip.conf`
with this:

```ini
[global]
index-url = http://localhost:3141/root/pypi
```

Following installations should then be going through your local cache.


## Set Up Local Index

By default devpi creates an index called `root/pypi`, which serves as a proxy
and cache for [PyPI][4], and you can’t upload your own packages to it.

In order to upload local packages we need to create another index, and to make
our lives easier we will also configure it so that it "inherits" from the
`root/pypi` index. What this meas is that if our search for a package in the
local index fails it will fall back to the `root/pypi` index and try to find it
there. Thus we can add all our private packages without losing the ability to
find all the public ones.

To achieve this we will start the `devpi-client` container as root (we need
to be able to make modifications):

```bash
docker run -it --rm --network host \
    -e DEVPI_USER=root \
    -e DEVPI_PASSWORD=password \
    -v $(pwd)/data-client:/devpi/client \
    loveyu/devpi-client:latest
```

Inside the container we will first create another user:

```bash
devpi user -c local password=something_long_and_complicated
```

Then we will create an index under this new user that inherits from the
`root/pypi` index:

```bash
devpi index -c local/stable bases=root/pypi volatile=False
```

Restart the container again, but this time specify the new index URL and the
new user:

```bash
docker run -it --rm --network host \
    -e DEVPI_URL="http://localhost:3141/local/stable" \
    -e DEVPI_USER=local \
    -e DEVPI_PASSWORD=something_long_and_complicated \
    -v $(pwd)/data-client:/devpi/client \
    loveyu/devpi-client:latest
```

You can now upload files from the `/devpi/client` folder to the current index
by running something similar to this:

```bash
devpi upload /devpi/client private_package-0.1.0-py3-none-any.whl
```

After this you should be able to install it by pointing to the new index:

```bash
pip install -i http://localhost:3141/local/stable private_package
```


## Upgrading

> :warning: This is semi-experimental since I am having trouble finding
> official documentation on the proper way to do this.

The best guide on upgrading devpi server I could find was [this one][9], so
I tried to integrate that into the entrypoint. It is not very automated, but
you will at least have full control of what it is doing the entire time.

> :information_source: You should also only have to do this procedure when
> there is a change in the database schema. Such a change [will warrant][10] a
> **major** version bump, so this process is not necessary if you are just
> upgrading a **minor** or a **patch** version.

Begin by stopping any of the currently running devpi containers, and then run
the same image again with the `/export` folder mounted to initiate an export
process:

```bash
docker run -it --rm -e DEVPI_PASSWORD=password \
    -v $(pwd)/data-server:/devpi/server \
    -v $(pwd)/tmp:/export \
    loveyu/devpi-server:old-tag
```

If this is successful you should now rename the old data folder (don't delete
it before you know the new one works):

```bash
sudo mv data-server data-server.bak
```

Run the new image with the `/import` folder mounted in order to initiate an
import process:

```bash
docker run -it --rm -e DEVPI_PASSWORD=password \
    -v $(pwd)/data-server:/devpi/server \
    -v $(pwd)/tmp:/import \
    loveyu/devpi-server:new-tag
```

When this one completes you can go back to running the image normally without
any of the `import/export` folders mounted. This should most likely give you a
functional upgraded instance of devpi. Cleanup of the extra folders can be done
with this simple command

```bash
sudo rm -r data-server.bak && sudo rm -r tmp
```


# Further Reading

I got most of the information I needed to complete this project from the
following sources, perhaps they are useful for you too:

- [Devpi Quickstart Guide](https://devpi.net/docs/devpi/devpi/stable/+d/quickstart-server.html)
- [Stefan Scherfke: Getting started with devpi](https://stefan.sofa-rockers.org/2017/11/09/getting-started-with-devpi/)
- [@kyhau: devpiNotes.md](https://gist.github.com/kyhau/0b54386fe220877310b9)
- [Mpho Mphego: How I Setup A Private Local PyPI Server Using Docker And Ansible](https://blog.mphomphego.co.za/blog/2021/06/15/How-I-setup-a-private-PyPI-server-using-Docker-and-Ansible.html)
- [@kyhau: devpiServerUpgrade.md][9]








[1]: https://devpi.net/docs/devpi/devpi/stable/+d/userman/devpi_commands.html#devpi-command-reference-server
[2]: https://devpi.net/docs/devpi/devpi/stable/+d/userman/devpi_commands.html
[3]: https://doc.devpi.net
[4]: https://pypi.org/
[5]: https://pypi.org/project/devpi-server/#history
[6]: https://pypi.org/project/devpi-client/#history
[7]: https://hub.docker.com/repository/docker/loveyu/devpi-server/tags?page=1&ordering=last_updated
[8]: https://hub.docker.com/repository/docker/loveyu/devpi-client/tags?page=1&ordering=last_updated
[9]: https://gist.github.com/kyhau/7707c6dfa25c2e14e345
[10]: https://github.com/devpi/devpi/issues/439#issuecomment-1064329177
