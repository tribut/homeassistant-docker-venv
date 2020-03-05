# Home Assistant using venv in Docker

*Run Home Assistant as non-root using the official docker image!*

There are many reasons why one would want to run [Home Assistant] as a different user than root, even when using docker:
When using a mapped volume for the configuration, you might want to access and edit it outside of the docker container - running the container as root will lead to permission problems. It is also a security best-practice, as it provides more protection against container escape exploits.

The Home Assistant docker image since version 0.94 installs additional required python packages site-wide, which only root is allowed to do. This simple script allows you to start Home Assistant inside the official image, but using a virtual python environment in which ordinary users are allowed to install packages.

Also, some packages on Alpine (homeassistant's base image) are buggy if not running as root (like ping). For that reason, the script also supports using a custom entrypoint to alter packages before dropping privileges.

## Usage

Follow the [official docs] for installation inside docker. But before you start your container, make sure the scripts `entrypoint` and `run` are available in `/config/docker`:

    cd /PATH_TO_YOUR_CONFIG
    git clone https://github.com/tribut/homeassistant-docker-venv docker

Then just pass the appropriate `--user` flag and the script to the `docker run` command:

    docker run --init -d --name="home-assistant" -e "TZ=America/New_York" -v /PATH_TO_YOUR_CONFIG:/config --net=host --user 1000:1000 homeassistant/home-assistant:stable /config/docker/run

If the custom entrypoint is wanted, pass the appropriate `PUID` and `PGID` environment variables and the scripts to the `docker run` command:

    docker run --init -d --name="home-assistant" -e "TZ=America/New_York" -e "PUID=1000" -e "PGID=1000" -e "PACKAGES=iputils" -v /PATH_TO_YOUR_CONFIG:/config --net=host --entrypoint=/config/docker/entrypoint homeassistant/home-assistant:stable

You must make sure the user you select can **write to your configuration directory** (`/PATH_TO_YOUR_CONFIG`) and has **access to your additional devices** (if applicable, anything made available using `docker run [...] --device`)!

### Default file permissions

If you need the files created by Home Assistant to be writeable by the group or everyone, you can customize the default permissions using the environment variable `UMASK` (see [umask (Wikipedia)](https://en.wikipedia.org/wiki/Umask) for an explanation).

For example, to allow write access for the group, add `-e UMASK=007` to your `docker run` call.

### Docker Compose

If you are running Docker Compose, you have to add these parameters to your `docker-compose.yml` file instead:

    user: '1000:1000'
    command: '/config/docker/run'
    environment:
      - UMASK=007

Or with the custom entrypoint:

    entrypoint: '/config/docker/entrypoint'
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=007

## Support

If you have any trouble with this setup, please open a [ticket] here. This configuration is [unsupported by the Home Assistant project](https://github.com/home-assistant/home-assistant/issues/24397#issuecomment-527446679).

[Home Assistant]: https://www.home-assistant.io/
[official docs]: https://www.home-assistant.io/docs/installation/docker/
[ticket]: https://github.com/tribut/homeassistant-docker-venv/issues
