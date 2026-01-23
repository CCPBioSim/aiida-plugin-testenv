FROM ghcr.io/mamba-org/micromamba:2.4.0-ubuntu24.04

LABEL maintainer="James Gebbie-Rayet <james.gebbie@stfc.ac.uk>"

ARG AIIDAVERSION=2.6.1
ARG AMBERTOOLSVERSION=24.8
ARG PYTHONVERSION=3.11
ARG NEW_MAMBA_USER=aiida
ARG NEW_MAMBA_USER_ID=1001
ARG NEW_MAMBA_USER_GID=121
USER root

RUN if grep -q '^ID=alpine$' /etc/os-release; then \
      # alpine does not have usermod/groupmod
      apk add --no-cache --virtual temp-packages shadow; \
    fi && \
    usermod "--login=${NEW_MAMBA_USER}" "--home=/home/${NEW_MAMBA_USER}" \
        --move-home "-u ${NEW_MAMBA_USER_ID}" "${MAMBA_USER}" && \
    groupmod "--new-name=${NEW_MAMBA_USER}" \
        "-g ${NEW_MAMBA_USER_GID}" "${MAMBA_USER}" && \
    if grep -q '^ID=alpine$' /etc/os-release; then \
      # remove the packages that were only needed for usermod/groupmod
      apk del temp-packages; \
    fi && \
    # Update the expected value of MAMBA_USER for the
    # _entrypoint.sh consistency check.
    echo "${NEW_MAMBA_USER}" > "/etc/arg_mamba_user" && \
    :
ENV MAMBA_USER=$NEW_MAMBA_USER
USER $MAMBA_USER

# Install dependencies
RUN micromamba install -c conda-forge python=$PYTHONVERSION aiida-core=$AIIDAVERSION aiida-core.services=$AIIDAVERSION && \
    micromamba install -c ambertools=$AMBERTOOLSVERSION && \
    micromamba clean --all --yes
