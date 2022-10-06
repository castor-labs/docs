FROM alpine:edge as base

ARG UID=1000
ARG USER=castor

RUN echo "https://dl-cdn.alpinelinux.org/alpine/edge/community" >> /etc/apk/repositories

# Create a development user
RUN adduser $USER --disabled-password --home /$USER --uid $UID

# Install Dependencies
RUN apk add --no-cache curl php-common php-cli mdbook

FROM base as dev

CMD ["mdbook", "watch"]

FROM base as dist