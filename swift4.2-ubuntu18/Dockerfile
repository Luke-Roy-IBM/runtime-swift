# build go proxy from source
FROM golang:1.15 AS builder_source
ARG GO_PROXY_GITHUB_USER=apache
ARG GO_PROXY_GITHUB_BRANCH=master
RUN git clone --branch ${GO_PROXY_GITHUB_BRANCH} \
   https://github.com/${GO_PROXY_GITHUB_USER}/openwhisk-runtime-go /src ;\
   cd /src ; env GO111MODULE=on CGO_ENABLED=0 go build main/proxy.go && \
   mv proxy /bin/proxy

# or build it from a release
FROM golang:1.15 AS builder_release
ARG GO_PROXY_RELEASE_VERSION=1.15@1.17.0
RUN curl -sL \
  https://github.com/apache/openwhisk-runtime-go/archive/{$GO_PROXY_RELEASE_VERSION}.tar.gz\
  | tar xzf -\
  && cd openwhisk-runtime-go-*/main\
  && GO111MODULE=on go build -o /bin/proxy

# Swift Part
FROM ubuntu:18.04
LABEL Description="Docker Container for the Swift programming language"

# Install related packages and set LLVM 3.9 as the compiler
RUN export DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true && apt-get -q update && \
    apt-get -q install -y \
    make \
    libc6-dev \
    clang-3.9 \
    curl \
    libedit-dev \
    libpython2.7 \
    libicu-dev \
    libssl-dev \
    libxml2 \
    tzdata \
    git \
    libcurl4-openssl-dev \
    zlib1g-dev \
    pkg-config \
    && update-alternatives --quiet --install /usr/bin/clang clang /usr/bin/clang-3.9 100 \
    && update-alternatives --quiet --install /usr/bin/clang++ clang++ /usr/bin/clang++-3.9 100 \
    && rm -r /var/lib/apt/lists/*

# Everything up to here should cache nicely between Swift versions, assuming dev dependencies change little
ARG SWIFT_PLATFORM=ubuntu18.04
ARG SWIFT_BRANCH=swift-4.2.4-release
ARG SWIFT_VERSION=swift-4.2.4-RELEASE

ENV SWIFT_PLATFORM=$SWIFT_PLATFORM \
    SWIFT_BRANCH=$SWIFT_BRANCH \
    SWIFT_VERSION=$SWIFT_VERSION

# Download GPG keys, signature and Swift package, then unpack, cleanup and execute permissions for foundation libs
RUN SWIFT_URL=https://swift.org/builds/$SWIFT_BRANCH/$(echo "$SWIFT_PLATFORM" | tr -d .)/$SWIFT_VERSION/$SWIFT_VERSION-$SWIFT_PLATFORM.tar.gz \
    && curl -fSsL $SWIFT_URL -o swift.tar.gz \
    && curl -fSsL $SWIFT_URL.sig -o swift.tar.gz.sig \
    && export GNUPGHOME="$(mktemp -d)" \
    #&& set -e; \
       # for key in \
      # pub   rsa4096 2017-11-07 [SC] [expires: 2019-11-07]
      # 8513444E2DA36B7C1659AF4D7638F1FB2B2B08C4
      # uid           [ unknown] Swift Automatic Signing Key #2 <swift-infrastructure@swift.org>
          #8513444E2DA36B7C1659AF4D7638F1FB2B2B08C4 \
      # pub   4096R/91D306C6 2016-05-31 [expires: 2018-05-31]
      #       Key fingerprint = A3BA FD35 56A5 9079 C068  94BD 63BC 1CFE 91D3 06C6
      # uid                  Swift 3.x Release Signing Key <swift-infrastructure@swift.org>
         # A3BAFD3556A59079C06894BD63BC1CFE91D306C6 \
      # pub   4096R/71E1B235 2016-05-31 [expires: 2019-06-14]
      #       Key fingerprint = 5E4D F843 FB06 5D7F 7E24  FBA2 EF54 30F0 71E1 B235
      # uid                  Swift 4.x Release Signing Key <swift-infrastructure@swift.org>
        #  5E4DF843FB065D7F7E24FBA2EF5430F071E1B235 \
       # ; do \
      #    gpg --quiet --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
     #   done \
    #&& gpg --batch --verify --quiet swift.tar.gz.sig swift.tar.gz \
    && tar -xzf swift.tar.gz --directory / --strip-components=1 \
    && rm -r "$GNUPGHOME" swift.tar.gz.sig swift.tar.gz \
    && chmod -R o+r /usr/lib/swift

# Print Installed Swift Version
RUN swift --version; sleep 5



#---------------------------------------------------------------------------------------------------------------------------
# Openwhisk Runtime 

# select the builder to use
ARG GO_PROXY_BUILD_FROM=release

RUN rm -rf /var/lib/apt/lists/* && apt-get clean && apt-get update \
	&& apt-get install -y --no-install-recommends locales python3 vim \
	&& rm -rf /var/lib/apt/lists/* \
	&& locale-gen en_US.UTF-8

ENV LANG="en_US.UTF-8" \
	LANGUAGE="en_US:en" \
	LC_ALL="en_US.UTF-8"

RUN mkdir -p /swiftAction
WORKDIR /swiftAction


COPY --from=builder_source /bin/proxy /bin/proxy_source
COPY --from=builder_release /bin/proxy /bin/proxy_release
RUN mv /bin/proxy_${GO_PROXY_BUILD_FROM} /bin/proxy
ADD swiftbuild.py /bin/compile
ADD swiftbuild.py.launcher.swift /bin/compile.launcher.swift
COPY _Whisk.swift /swiftAction/Sources/
COPY Package.swift /swiftAction/
COPY buildandrecord.py /swiftAction/
COPY main.swift /swiftAction/Sources/
RUN swift build -c release; \
	touch /swiftAction/Sources/main.swift; \
	rm /swiftAction/.build/release/Action; \
    chmod +x /swiftAction/buildandrecord.py; \
	/swiftAction/buildandrecord.py

RUN chmod +x /bin/compile && chmod +x /bin/proxy
ENV OW_COMPILER=/bin/compile

ENTRYPOINT [ "/bin/proxy" ]

COPY _Whisk.swift /swiftAction/Sources/
COPY Package.swift /swiftAction/
RUN apt-get update \
    # Update installed packages to get security fixes if available.
    && apt-get upgrade -y --no-install-recommends --with-new-pkgs\
    # Cleanup apt data, we do not need them later on.
    && rm -rf /var/lib/apt/lists/* \
    && swift build -c release \
    && touch /swiftAction/Sources/main.swift \
    && rm /swiftAction/.build/release/Action \
    && /swiftAction/buildandrecord.py