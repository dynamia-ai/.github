# Copy these targets into the adopting repository's Makefile. Adjust the context,
# Dockerfile, image reference, and labels for that repository.
DOCKER ?= docker
IMAGE_CONTEXT ?= .
DOCKERFILE ?= Dockerfile
IMAGE_REF ?= local/validation:dev
IMAGE_PLATFORM ?= linux/amd64
IMAGE_SOURCE ?= local
IMAGE_REVISION ?= dev

define build_image
$(DOCKER) buildx build \
	--file "$(DOCKERFILE)" \
	--platform "$(1)" \
	--tag "$(2)" \
	--label "org.opencontainers.image.source=$(IMAGE_SOURCE)" \
	--label "org.opencontainers.image.revision=$(IMAGE_REVISION)" \
	--load \
	"$(IMAGE_CONTEXT)"
endef

.PHONY: build-image build-image-amd64 build-image-arm64

build-image:
	$(call build_image,$(IMAGE_PLATFORM),$(IMAGE_REF))

build-image-amd64:
	$(call build_image,linux/amd64,$(IMAGE_REF)-amd64)

build-image-arm64:
	$(call build_image,linux/arm64,$(IMAGE_REF)-arm64)
