#!/usr/bin/with-contenv bashio

USER="homeassistant"
GROUP="$USER"
PUID="${PUID:-1000}"
PGID="${PGID:-1000}"

UMASK="${UMASK:-}"

PACKAGES="${PACKAGES:-}"

VENV_PATH="${VENV:-/var/tmp/homeassistant-venv}"
CONFIG_PATH=/config

#
# Create user
#

# Some HA commands seem to fail if we don't have an actual user.
# ie: shell_command would return error code 255
bashio::log.info "Creating user $USER with $PUID:$PGID"

deluser "$USER" >/dev/null 2>&1 || true
delgroup "$GROUP" >/dev/null 2>&1 || true

# Re-use existing group (can't delgroup a group that is in use)
group="$(getent group "$PGID" | cut -d: -f1 || true)"
if [ -z "$group" ]; then
  addgroup -g "$PGID" "$GROUP"
else
  bashio::log.notice "Re-using existing group with gid $PGID: $group"
  GROUP="$group"
fi

# Replace existing user (ensures correct shell and primary group)
user="$(getent passwd "$PUID" | cut -d: -f1 || true)"
if [ -n "$user" ]; then
  bashio::log.notice "Replacing existing user with uid $PUID: $user"
  deluser "$user"
fi
adduser -G "$GROUP" -D -H -u "$PUID" "$USER"

if [ -n "${EXTRA_GID:-}" ]; then
  bashio::log.info "Resolving suppementary GIDs: $EXTRA_GID"
  supplementary_groups=()

  for gid in $EXTRA_GID; do
    group="$(getent group "$gid" | cut -d: -f1 || true)"

    if [ -z "$group" ]; then
      group="$USER-$gid"
      addgroup -g "$gid" "$group"
    fi

    supplementary_groups+=( "$group" )
  done

  bashio::log.info "Appending supplementary groups: ${supplementary_groups[@]}"
  for group in "${supplementary_groups[@]}"; do
    addgroup "$USER" "$group"
  done
fi

#
# Install extra packages
#

if [ -n "${PACKAGES}" ]; then
  bashio::log.info "Installing extra packages: $PACKAGES"
  apk add --quiet --no-progress --no-cache -- $PACKAGES
fi

#
# Create virtual environment
#

bashio::log.info "Initializing venv in $VENV_PATH"
su "$USER" \
  -c "python3 -m venv --system-site-packages '$VENV_PATH'"

#
# Fix permissions
#

if [ -n "${UMASK}" ]; then
  bashio::log.info "Setting umask: $UMASK"
  umask "$UMASK"
fi

#
# Run homeassistant
#

bashio::log.info "Activating venv"
. "$VENV_PATH/bin/activate"

# Everything below should be kept in sync with
#   core:rootfs/etc/services.d/home-assistant/run
# from upstream:
cd "$CONFIG_PATH" || bashio::exit.nok "Can't find config folder: $CONFIG_PATH"

# Enable Jemalloc for Home Assistant Core, unless disabled
if [[ -z "${DISABLE_JEMALLOC+x}" ]]; then
  export LD_PRELOAD="/usr/local/lib/libjemalloc.so.2"
fi

bashio::log.info "Starting homeassistant"
exec \
  s6-setuidgid "$USER" \
  python3 -m homeassistant --config "$CONFIG_PATH"
