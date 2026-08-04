# Copy this target into the adopting repository's Makefile, then add assertions
# for required binaries, assets, configuration, and a harmless version command.
DOCKER ?= docker
IMAGE_REF ?=
IMAGE_SMOKE_COMMAND ?=

.PHONY: verify-image

verify-image:
	@test -n "$(strip $(IMAGE_REF))" || { \
		echo "IMAGE_REF is required" >&2; \
		exit 1; \
	}
	$(DOCKER) image inspect "$(IMAGE_REF)" >/dev/null
	@test -n "$(strip $(IMAGE_SMOKE_COMMAND))" || { \
		echo "IMAGE_SMOKE_COMMAND must run a harmless repository-specific check" >&2; \
		exit 1; \
	}
	$(IMAGE_SMOKE_COMMAND)
