# docker build -f olric-debug/Dockerfile -t olricio/olric-debug:$TAG .
# docker push olricio/olric-debug:$TAG

FROM golang:alpine as build
RUN apk add build-base
RUN GO111MODULE=on go get -ldflags="-s -w" github.com/buraksezer/olric/cmd/olric-cli@v0.4.0-rc.1
RUN GO111MODULE=on go get -ldflags="-s -w" github.com/buraksezer/olric/cmd/olric-stats@v0.4.0-rc.1
RUN GO111MODULE=on go get -ldflags="-s -w" github.com/buraksezer/olric/cmd/olric-benchmark@v0.4.0-rc.1