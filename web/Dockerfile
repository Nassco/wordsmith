# Stage 1 : compilation
FROM golang:1.21-alpine AS build
WORKDIR /app
COPY go.mod* go.sum* ./
RUN go mod download 2>/dev/null || true
COPY . .
RUN CGO_ENABLED=0 go build dispatcher.go

# Stage 2 : image finale minimale
FROM scratch
COPY --from=build /app/dispatcher /dispatcher
COPY --from=build /app/static /static
EXPOSE 80
CMD ["/dispatcher"]