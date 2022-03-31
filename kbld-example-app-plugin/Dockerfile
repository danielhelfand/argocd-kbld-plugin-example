FROM golang:1.15

RUN mkdir /app

COPY . /app

WORKDIR /app

RUN go build -o go-web-server

EXPOSE 8080

USER 1001

CMD ["./go-web-server"]