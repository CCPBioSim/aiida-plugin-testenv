ARG BASE_IMAGE=ubuntu:24.04
FROM $BASE_IMAGE

LABEL maintainer="James Gebbie-Rayet <james.gebbie@stfc.ac.uk>"

# Define customisable args
ARG AMBERTOOLS_DL=null
ARG AMBERTOOLSVERSION=25
ARG SYSTEM_USER=aiida
ARG SYSTEM_UID=1001
ARG SYSTEM_GID=121

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Set OS level deps
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update --yes && \
    apt-get upgrade --yes && \
    apt-get install --yes --no-install-recommends \
    bison \
    build-essential \
    bzip2 \
    ca-certificates \
    cmake \
    curl \
    flex \
    erlang \
    gfortran \
    git \
    libbz2-dev \
    locales \
    openssh-client \
    sudo \
    vim \
    wget \
    xz-utils \
    zlib1g-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# Configure env vars
ENV CONDA_DIR=/opt/conda \
    SHELL=/bin/bash \
    SYSTEM_USER="${SYSTEM_USER}" \
    SYSTEM_UID=${SYSTEM_UID} \
    SYSTEM_GID=${SYSTEM_GID} \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8
ENV PATH="${CONDA_DIR}/bin:${PATH}" \
    HOME="/home/${SYSTEM_USER}"

# Create SYSTEM_USER with name aiida  with UID=1001 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    addgroup --gid "${SYSTEM_GID}" "${SYSTEM_USER}" && \
    useradd -l -m -s /bin/bash -N --uid "${SYSTEM_UID}" --gid "${SYSTEM_GID}" "${SYSTEM_USER}" && \
    mkdir -p "${CONDA_DIR}" && \
    chown "${SYSTEM_USER}:${SYSTEM_GID}" "${CONDA_DIR}" "${HOME}" && \
    chmod g+w /etc/passwd

# make a github actions mount point.
RUN mkdir /github && \
    chown "${SYSTEM_USER}:${SYSTEM_GID}" /github

WORKDIR /tmp
RUN wget --no-verbose $AMBERTOOLS_DL/ambertools$AMBERTOOLSVERSION.tar.bz2 && \
    tar xjf ambertools$AMBERTOOLSVERSION.tar.bz2 && \
    rm ambertools$AMBERTOOLSVERSION.tar.bz2 && \
    mv ambertools${AMBERTOOLSVERSION}_src ambertools_src

WORKDIR /tmp/ambertools_src
RUN mkdir /tmp/build
WORKDIR /tmp/build

RUN cmake /tmp/ambertools_src -DCMAKE_INSTALL_PREFIX=/opt/ambertools -DUSE_CONDA_LIBS=FALSE -DBUILD_PYTHON=FALSE -DCMAKE_INSTALL_RPATH_USE_LINK_PATH=TRUE -DDOWNLOAD_MINICONDA=FALSE -DCOMPILER=MANUAL -DBUILD_GUI=FALSE -DCOMPILER=GNU -DOPENMP=TRUE -DCUDA=FALSE -DMPI=FALSE -DUSE_FFT=True -DBUILD_DEPRECATED=False -DBUILD_INDEV=False -DBUILD_PERL=True -DOPTIMIZE=True
RUN make -j8
RUN make install

# Cleanup.
RUN rm -r /tmp/ambertools_src /tmp/build

# Add the exec to the path.
ENV AMBERHOME=/opt/ambertools

USER ${SYSTEM_UID}

# Download and install Micromamba, and initialize Conda prefix.
COPY --chown="${SYSTEM_UID}:${SYSTEM_GID}" amber/.condarc "${CONDA_DIR}/.condarc"
WORKDIR /tmp
RUN set -x && \
    arch=$(uname -m) && \
    if [ "${arch}" = "x86_64" ]; then \
        # Should be simpler, see <https://github.com/mamba-org/mamba/issues/1437>
        arch="64"; \
    fi && \
    wget --progress=dot:giga -O /tmp/micromamba.tar.bz2 \
        "https://micromamba.snakepit.net/api/micromamba/linux-${arch}/latest" && \
    tar -xvjf /tmp/micromamba.tar.bz2 --strip-components=1 bin/micromamba && \
    rm /tmp/micromamba.tar.bz2 && \
    ./micromamba install \
        --root-prefix="${CONDA_DIR}" \
        --prefix="${CONDA_DIR}" \
        --yes \
        'mamba' && \
    rm micromamba

# Add ~/.local/bin to PATH where the dependencies get installed via pip
# This require the package installed with `--user` flag in pip
ENV PATH=${PATH}:/home/${SYSTEM_USER}/.local/bin

# Switch back to USER aiida to avoid accidental container runs as root
USER ${SYSTEM_UID}
WORKDIR "${HOME}"
