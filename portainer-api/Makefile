UPSTREAM_REPO ?= git@github.com:portainer/portainer-api-docs.git
UPSTREAM_DIR  := upstream
SPEC_DIR       = references/spec

.PHONY: specs

# Refresh references/spec/ from upstream (EE — the edition the skill bundles)
# Usage: make specs VERSION=2.42.0
specs:
	@if [ -z "$(VERSION)" ]; then \
		echo "VERSION is required, e.g. make specs VERSION=2.42.0" >&2; \
		exit 1; \
	fi
	@if [ -d $(UPSTREAM_DIR)/.git ]; then \
		git -C $(UPSTREAM_DIR) fetch --depth=1 origin HEAD && \
		git -C $(UPSTREAM_DIR) reset --hard FETCH_HEAD; \
	else \
		git clone --depth=1 --filter=blob:none --no-checkout \
			$(UPSTREAM_REPO) $(UPSTREAM_DIR); \
	fi
	git -C $(UPSTREAM_DIR) sparse-checkout set --no-cone /versions/ee/$(VERSION)/
	git -C $(UPSTREAM_DIR) checkout
	rm -rf $(SPEC_DIR) && mkdir -p $(SPEC_DIR)
	cp $(UPSTREAM_DIR)/versions/ee/$(VERSION)/*.yaml $(SPEC_DIR)/
	uv run --with pyyaml scripts/postprocess_specs.py $(SPEC_DIR) \
		--version $(VERSION)
