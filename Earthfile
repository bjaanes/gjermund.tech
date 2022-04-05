VERSION 0.6
FROM klakegg/hugo:ext-alpine
WORKDIR /app


build:
    COPY . .
    RUN hugo
    SAVE ARTIFACT public

docker:
    ARG VERSION
    FROM joseluisq/static-web-server:latest
    COPY +build/public ./public
    CMD ["-p", "8080", "-g", "info"]
    SAVE IMAGE --push empowergjermund/gjermund.tech:$VERSION empowergjermund/gjermund.tech:latest