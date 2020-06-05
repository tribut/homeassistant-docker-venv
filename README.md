# Home Assistant using venv in Docker

*Run Home Assistant as non-root using the official docker image!*

There are many reasons why one would want to run [Home Assistant] as a different user than root, even when using docker:
When using a mapped volume for the configuration, you might want to access and edit it outside of the docker container - running the container as root will lead to permission problems. It is also a security best-practice, as it provides more protection against container escape exploits.

The Home Assistant docker image since version 0.94 installs additional required python packages site-wide, which only root is allowed to do. This simple script allows you to start Home Assistant inside the official image, but using a virtual python environment in which ordinary users are allowed to install packages.

Since version 0.107, the Home Assistant docker image uses the [S6-Overlay](https://github.com/just-containers/s6-overlay) as its init system. S6 _requires_ root to start, privilege drop is performed right before starting the Home Assistant service.

Finally, some packages on Alpine (Home Assistant's base image) are buggy if not running as root (like ping). For that reason, the custom run script supports installing extra packages, specified in the `PACKAGES` environment variable, before starting Home Assistant.

## Usage

Follow the [official docs] for installation inside docker. But before you start your container, make sure the script `run` is available in `/config/docker`:

    cd /PATH_TO_YOUR_CONFIG
    git clone https://github.com/tribut/homeassistant-docker-venv docker

Then you must add three things to your `docker run` command:

1. The `PUID` environment variable will represent the user id under which Home Assistant runs.
1. The `PGID` environment variable will represent the group id under which Home Assistant runs.
1. Volume mount the `run` script on top of Home Assistant's run script: `/etc/services.d/home-assistant/run`

The resulting `docker run` command would look like:

```sh
docker run -d \
    --name="home-assistant" \
    -e "TZ=America/New_York" \
    -e "PUID=1000" \
    -e "PGID=1000" \
    -e "PACKAGES=iputils" \
    -v "/PATH_TO_YOUR_CONFIG:/config" \
    -v "/PATH_TO_YOUR_CONFIG/docker/run:/etc/services.d/home-assistant/run" \
    --net=host \
    homeassistant/home-assistant:stable
```

You must make sure the user you select can **write to your configuration directory** (`/PATH_TO_YOUR_CONFIG`) and has **access to your additional devices** (if applicable, anything made available using `docker run [...] --device`)!

**Do not use the --init flag. S6 is the init system. Using that flag would likely prevent Home Assistant from starting.**

### Default file permissions

If you need the files created by Home Assistant to be writeable by the group or everyone, you can customize the default permissions using the environment variable `UMASK` (see [umask (Wikipedia)](https://en.wikipedia.org/wiki/Umask) for an explanation).

For example, to allow write access for the group, add `-e UMASK=007` to your `docker run` call.

### Docker Compose

If you are running Docker Compose, you have to add these parameters to your `docker-compose.yml` file instead:

```yaml
home-assistant:
    container_name: home-assistant
    image: "homeassistant/home-assistant:stable"
    network_mode: host
    environment:
        - TZ=America/New_York
        - PUID=1000
        - PGID=1000
        - UMASK=007
        - PACKAGES=iputils
    volumes:
        - "/PATH_TO_YOUR_CONFIG/config:/config"
        - "/PATH_TO_YOUR_CONFIG/docker/run:/etc/services.d/home-assistant/run"
```

## Support

If you have any trouble with this setup, please open a [ticket] here. This configuration is [unsupported by the Home Assistant project](https://github.com/home-assistant/home-assistant/issues/24397#issuecomment-527446679).

[Home Assistant]: https://www.home-assistant.io/
[official docs]: https://www.home-assistant.io/docs/installation/docker/
[ticket]: https://github.com/tribut/homeassistant-docker-venv/issues
