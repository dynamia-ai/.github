# Copy these targets into the adopting repository's Makefile. Replace CHART_PATH
# and add representative render assertions for the project's supported values.
HELM ?= helm
HELM_DOCS ?= helm-docs

HELM_VERSION ?= v4.2.3
HELM_SCHEMA_VERSION ?= 2.5.0

CHART_PATH ?= charts/REPLACE_ME
HELM_RELEASE_NAME ?= chart-validation
HELM_TEMPLATE_ARGS ?=

.PHONY: check-chart-tools update-chart-docs verify-helm-chart

check-chart-tools:
	@test "$$($(HELM) version --template '{{.Version}}')" = "$(HELM_VERSION)" || { \
		echo "Helm version mismatch; expected $(HELM_VERSION)" >&2; \
		exit 1; \
	}
	@command -v "$(HELM_DOCS)" >/dev/null || { \
		echo "helm-docs is required" >&2; \
		exit 1; \
	}
	@test "$$($(HELM) plugin list | awk '$$1 == "schema" { print $$2; exit }')" = "$(HELM_SCHEMA_VERSION)" || { \
		echo "helm schema plugin version mismatch; expected $(HELM_SCHEMA_VERSION)" >&2; \
		exit 1; \
	}

update-chart-docs: check-chart-tools
	cd "$(CHART_PATH)" && "$(HELM_DOCS)" \
		--chart-search-root . \
		--chart-to-generate . \
		--skip-version-footer
	cd "$(CHART_PATH)" && "$(HELM)" schema \
		-f values.yaml \
		--use-helm-docs \
		--indent 2

verify-helm-chart: check-chart-tools
	git ls-files --error-unmatch \
		"$(CHART_PATH)/README.md" \
		"$(CHART_PATH)/values.schema.json" >/dev/null
	@set -eu; \
	tmp_dir="$$(mktemp -d)"; \
	cp "$(CHART_PATH)/README.md" "$$tmp_dir/README.md"; \
	cp "$(CHART_PATH)/values.schema.json" "$$tmp_dir/values.schema.json"; \
	restore() { \
		cp "$$tmp_dir/README.md" "$(CHART_PATH)/README.md"; \
		cp "$$tmp_dir/values.schema.json" "$(CHART_PATH)/values.schema.json"; \
		rm -rf "$$tmp_dir"; \
	}; \
	trap restore EXIT HUP INT TERM; \
	$(MAKE) update-chart-docs >/dev/null; \
	diff -u "$$tmp_dir/README.md" "$(CHART_PATH)/README.md"; \
	diff -u "$$tmp_dir/values.schema.json" "$(CHART_PATH)/values.schema.json"; \
	restore; \
	trap - EXIT HUP INT TERM
	@if grep -Eq '^dependencies:' "$(CHART_PATH)/Chart.yaml"; then \
		git ls-files --error-unmatch "$(CHART_PATH)/Chart.lock" >/dev/null || { \
			echo "a tracked Chart.lock is required when Chart.yaml declares dependencies" >&2; \
			exit 1; \
		}; \
	fi
	$(HELM) dependency build "$(CHART_PATH)"
	@test -z "$$(git status --porcelain --untracked-files=all -- \
		"$(CHART_PATH)/Chart.lock" "$(CHART_PATH)/charts")" || { \
		echo "helm dependency build changed the committed dependency tree" >&2; \
		git status --short -- "$(CHART_PATH)/Chart.lock" "$(CHART_PATH)/charts" >&2; \
		exit 1; \
	}
	KUBECONFIG=/dev/null $(HELM) lint --strict "$(CHART_PATH)"
	KUBECONFIG=/dev/null $(HELM) template \
		"$(HELM_RELEASE_NAME)" \
		"$(CHART_PATH)" \
		$(HELM_TEMPLATE_ARGS) >/dev/null
	@set -eu; \
	package_dir="$$(mktemp -d)"; \
	trap 'rm -rf "$$package_dir"' EXIT HUP INT TERM; \
	$(HELM) package "$(CHART_PATH)" --destination "$$package_dir" >/dev/null; \
	set -- "$$package_dir"/*.tgz; \
	if [ "$$#" -ne 1 ] || [ ! -f "$$1" ]; then \
		echo "expected exactly one packaged Chart" >&2; \
		exit 1; \
	fi; \
	$(HELM) show chart "$$1" >/dev/null
