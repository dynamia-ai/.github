# Copy these targets into the adopting repository's Makefile and then adjust the
# package lists, build tags, CGO settings, and nested modules for that project.
GO ?= go
GOLANGCI_LINT ?= golangci-lint
GO_PACKAGES ?= ./...
GO_BUILD_PACKAGES ?= ./...
GO_COVERAGE_FILE ?= cover.out

.PHONY: fmt-check vet lint test verify-build

fmt-check:
	@files="$$(git ls-files -- '*.go' ':!:vendor/**')"; \
	if [ -z "$$files" ]; then \
		echo "no tracked Go files found" >&2; \
		exit 1; \
	fi; \
	unformatted="$$(gofmt -l $$files)"; \
	if [ -n "$$unformatted" ]; then \
		echo "Go files require formatting:" >&2; \
		printf '%s\n' "$$unformatted" >&2; \
		exit 1; \
	fi

vet:
	$(GO) vet $(GO_PACKAGES)

lint:
	$(GOLANGCI_LINT) run $(GO_PACKAGES)

test:
	$(GO) test -race -covermode=atomic -coverprofile="$(GO_COVERAGE_FILE)" $(GO_PACKAGES)
	test -s "$(GO_COVERAGE_FILE)"

verify-build:
	$(GO) build $(GO_BUILD_PACKAGES)
