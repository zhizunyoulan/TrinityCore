ARG UBUNTU_VERSION=18.04 # lts

# This target lays out the general directory skeleton for trinity,
# This target isn't intended to be directly used
FROM ubuntu:$UBUNTU_VERSION AS skeleton

# Note: ARG instructions defined after FROM are available in this build stage.
# Placing ARG TZ here (after FROM) ensures it is accessible for configuring the timezone below.
ARG TZ=Asia/Shanghai
ARG DOCKER=1
ARG DEBIAN_FRONTEND=noninteractive

ARG CONF_DIR=/trinity/dist/etc
ARG DIST_DIR=/trinity/dist

RUN mkdir -pv \
        /trinity/dist/etc \
        /trinity/ref/etc  \
        /trinity/data  \
        /trinity/logs  \
        /trinity/build 

# Configure Timezone and install necessary packages for building and running TrinityCore
RUN apt-get update                                                       \
    && apt-get install -y --no-install-recommends tzdata ca-certificates \
    && ln -snf "/usr/share/zoneinfo/$TZ" /etc/localtime                  \
    && echo "$TZ" > /etc/timezone                                        \
    && dpkg-reconfigure --frontend noninteractive tzdata                 \
    && apt-get install -y --no-install-recommends \
    libmysqlclient-dev  libbz2-dev libreadline-dev libncurses-dev p7zip \
    libboost-all-dev libssl-dev \
    && rm -rf /var/lib/apt/lists/*


WORKDIR /trinity


# This target builds the docker image
# This target can be useful to inspect the explicit outputs from the build,
FROM skeleton AS build

ARG CSCRIPTS="static"
ARG CSCRIPTS_DEFAULT_LINKAGE="static"
ARG CWITH_WARNINGS="ON"
ARG CMAKE_EXTRA_OPTIONS=""
ARG GIT_DISCOVERY_ACROSS_FILESYSTEM=1

ARG CCACHE_MAXSIZE="1000MB"
ARG CCACHE_SLOPPINESS="pch_defines,time_macros,include_file_mtime"
ARG CCACHE_COMPRESS=""
ARG CCACHE_COMPRESSLEVEL="9"
ARG CCACHE_COMPILERCHECK="content"
ARG CCACHE_LOGFILE=""

ARG CACHEBUST=1


COPY CMakeLists.txt PreLoad.cmake revision_data.h.in.cmake  /trinity/
COPY cmake /trinity/cmake
COPY contrib /trinity/contrib
COPY dep /trinity/dep
COPY src /trinity/src

RUN apt-get update \
     && apt-get install -y --no-install-recommends \
     git cmake make clang gcc g++ \
     && update-alternatives --install /usr/bin/cc cc /usr/bin/clang 100 \
     && update-alternatives --install /usr/bin/c++ c++ /usr/bin/clang 100 \
     && rm -rf /var/lib/apt/lists/*

WORKDIR /trinity/build

RUN  rm -rf /trinity/build/* \
    && cmake ../ \
          -DTOOLS_BUILD=1                                 \
          -DSCRIPTS="static"                              \
          -DWITH_WARNINGS=1                               \
          -DCONF_DIR="$CONF_DIR"                          \
    && make -j$(($(nproc) + 1))                           \
    && make install

#############################
# Base runtime for services #
#############################

FROM skeleton AS runtime

ARG USER_ID=1000
ARG GROUP_ID=1000
ARG DOCKER_USER=trinity

ENV trinity_COMPONENT=undefined

RUN groupadd --gid "$GROUP_ID"  "$DOCKER_USER" && \
    useradd -d /trinity --uid "$USER_ID"  --gid "$GROUP_ID"  "$DOCKER_USER" && \
    passwd -d "$DOCKER_USER" && \
    chown -R "$DOCKER_USER:$DOCKER_USER" /trinity


COPY --chown=$USER_ID:$GROUP_ID \
     --chmod=755 \
     contrib/Docker/entrypoint.sh /trinity/entrypoint.sh

RUN apt-get update \
     && apt-get install -y --no-install-recommends \
     mysql-client                                  \
     && rm -rf /var/lib/apt/lists/*

USER $DOCKER_USER

VOLUME /trinity/logs

WORKDIR /trinity/dist/bin


ENTRYPOINT ["/usr/bin/env", "bash", "/trinity/entrypoint.sh"]



###############
# Auth Server #
###############

FROM runtime AS bnetserver
LABEL description="trinity Auth Server"

ENV trinity_COMPONENT=bnetserver

COPY --chown=$USER_ID:$GROUP_ID \
     --from=build \
     /trinity/dist/etc/bnetserver.conf.dist /trinity/ref/etc/bnetserver.conf.dist

COPY --chown=$USER_ID:$GROUP_ID \
     --from=build \
     /usr/local/bin/bnetserver /trinity/dist/bin/bnetserver

COPY --chown=$USER_ID:$GROUP_ID \
     --from=build \
     /usr/local/bin/*.pem /trinity/dist/bin/

VOLUME $CONF_DIR

CMD ["/trinity/dist/bin/bnetserver"]

################
# World Server #
################

FROM runtime AS worldserver

LABEL description="trinity World Server"

ENV trinity_COMPONENT=worldserver


COPY --chown=$USER_ID:$GROUP_ID \
     --from=build \
     /trinity/dist/etc/worldserver.conf.dist /trinity/ref/etc/worldserver.conf.dist

COPY --chown=$USER_ID:$GROUP_ID \
     --from=build \
     /usr/local/bin/worldserver /trinity/dist/bin/worldserver

COPY --chown=$USER_ID:$GROUP_ID TDB_full_837.20101_2020_10_20.7z /trinity/

COPY --chown=$USER_ID:$GROUP_ID sql /trinity/sql

RUN p7zip -d /trinity/TDB_full_837.20101_2020_10_20.7z

VOLUME $CONF_DIR

VOLUME /trinity/data



CMD ["/trinity/dist/bin/worldserver"]