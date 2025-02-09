FROM golang:1.21 as build
COPY . /src
WORKDIR /src
RUN CGO_ENABLED=0 GOOS=linux go build -o hw

FROM scratch
COPY --from=build /src/hw .
CMD ["/hw"]