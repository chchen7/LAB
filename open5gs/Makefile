SHELL := /bin/bash

.PHONY: build

build:
	set -a; \
	source .env; \
	set +a; \
	cd base && docker build --no-cache --force-rm -t docker_open5gs .

	cd mongo && docker build --no-cache --force-rm -t docker_mongo .