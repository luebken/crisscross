FROM golang:1.16-alpine

WORKDIR /app

COPY go.mod ./
COPY go.sum ./
RUN go mod download

COPY *.go ./

RUN go build -o /nop-paaf

EXPOSE 8081

CMD [ "/nop-paaf" ]