FROM buildpack-deps:curl

ENV NODE_VERSION 24.2.0
ENV GOLANG_VERSION 1.24.4

RUN cd /usr/local/bin && \
  curl -L https://github.com/google/ko/releases/download/v0.18.0/ko_0.18.0_Linux_x86_64.tar.gz | tar xvz ko && \
  curl -L https://get.helm.sh/helm-v3.18.3-linux-amd64.tar.gz | tar xvz --strip-components=1 linux-amd64/helm

RUN cd /usr/local && \
  curl -L https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.gz | tar xz --strip-components=1 node-v$NODE_VERSION-linux-x64/lib node-v$NODE_VERSION-linux-x64/bin node-v$NODE_VERSION-linux-x64/include && \
  corepack enable && corepack prepare pnpm@10.12.2 --activate

ENV GOPATH /go

RUN curl -L https://golang.org/dl/go$GOLANG_VERSION.linux-amd64.tar.gz | tar xz -C /usr/local && \
  mkdir -p "$GOPATH/src" "$GOPATH/bin" && chmod -R 777 "$GOPATH"

RUN echo $(pnpm --version)

ENV PATH /usr/local/go/bin:$GOPATH/bin:$PATH
ENV GOROOT=/usr/local/go
WORKDIR $GOPATH

COPY build-app.sh /usr/local/bin