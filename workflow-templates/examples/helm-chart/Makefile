# Copy these targets into the adopting repository's Makefile. Replace CHART_PATH
# and set HELM_TEMPLATE_ARGS when the Chart needs non-default values to render.
HELM ?= helm
HELM_DOCS ?= helm-docs

CHART_PATH ?= charts/REPLACE_ME
HELM_RELEASE_NAME ?= chart-validation
HELM_TEMPLATE_ARGS ?=

.PHONY: update-chart-docs verify-helm-chart

update-chart-docs:
	cd "$(CHART_PATH)" && "$(HELM_DOCS)" \
		--chart-search-root . \
		--chart-to-generate . \
		--skip-version-footer
	cd "$(CHART_PATH)" && "$(HELM)" schema \
		-f values.yaml \
		--use-helm-docs \
		--indent 2

verify-helm-chart: update-chart-docs
	git diff --exit-code -- \
		"$(CHART_PATH)/README.md" \
		"$(CHART_PATH)/values.schema.json"
	$(HELM) lint "$(CHART_PATH)"
	$(HELM) template \
		"$(HELM_RELEASE_NAME)" \
		"$(CHART_PATH)" \
		$(HELM_TEMPLATE_ARGS) >/dev/null
