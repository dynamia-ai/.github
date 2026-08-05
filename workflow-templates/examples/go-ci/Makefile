# Copy these targets into the adopting repository's Makefile and then adjust the
# package lists, build tags, CGO settings, and nested modules for that project.
GO ?= go
GOLANGCI_LINT ?= golangci-lint
GOLANGCI_LINT_CONFIG ?= .golangci.yml
GO_PACKAGES ?= ./...
GO_BUILD_PACKAGES ?= ./...
GO_COVERAGE_FILE ?= cover.out

.PHONY: ci fmt fmt-check vet lint test verify-build

ci: fmt-check vet lint test verify-build

fmt:
	$(GOLANGCI_LINT) fmt --config "$(GOLANGCI_LINT_CONFIG)"

fmt-check:
	$(GOLANGCI_LINT) fmt --diff --config "$(GOLANGCI_LINT_CONFIG)"

vet:
	$(GO) vet $(GO_PACKAGES)

lint:
	$(GOLANGCI_LINT) run --config "$(GOLANGCI_LINT_CONFIG)" $(GO_PACKAGES)

test:
	$(GO) test -race -covermode=atomic -coverprofile="$(GO_COVERAGE_FILE)" $(GO_PACKAGES)
	test -s "$(GO_COVERAGE_FILE)"

verify-build:
	$(GO) build $(GO_BUILD_PACKAGES)
