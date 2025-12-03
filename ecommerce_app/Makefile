# Get the current directory (where Makefile is located)
CURRENT_DIR := $(shell pwd)

docker-build:
	docker build -t logstash-vespa:latest .

docker-test-config:
	@echo "Testing Logstash configuration..."
	@echo "Note: Connection errors are normal if Vespa isn't running - config validation will still check syntax"
	docker run --rm \
		-v $(CURRENT_DIR)/dataset/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro \
		logstash-vespa:latest \
		logstash --config.test_and_exit --config.debug --path.settings=/usr/share/logstash/config || true

docker-test-config-verbose:
	@echo "Testing Logstash configuration with verbose output..."
	docker run --rm -it \
		-v $(CURRENT_DIR)/dataset/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro \
		logstash-vespa:latest \
		logstash --config.test_and_exit --config.debug --log.level=debug --path.settings=/usr/share/logstash/config

docker-check-plugin:
	@echo "Checking if Vespa plugin is installed..."
	docker run --rm logstash-vespa:latest logstash-plugin list | grep vespa

docker-list-certs:
	@echo "Available Vespa certificate directories:"
	@find $$HOME/.vespa -name "data-plane-public-cert.pem" -type f 2>/dev/null | xargs dirname 2>/dev/null | while read dir; do \
		echo "  $$dir"; \
	done || echo "  (none found - run 'vespa auth cert' first)"

docker-run:
	@echo "Finding Vespa certificate directory..."
	@if [ -n "$$VESPA_CERT_DIR" ]; then \
		echo "Using VESPA_CERT_DIR from environment: $$VESPA_CERT_DIR"; \
	else \
		VESPA_CONFIG_OUTPUT=$$(vespa config get application 2>/dev/null || echo ""); \
		if [ -n "$$VESPA_CONFIG_OUTPUT" ]; then \
			VESPA_APP=$$(echo "$$VESPA_CONFIG_OUTPUT" | sed 's/^[^=]*= *//' | xargs); \
			VESPA_CERT_DIR=$$HOME/.vespa/$$VESPA_APP; \
			echo "Using application from vespa config: $$VESPA_APP"; \
			if [ ! -d "$$VESPA_CERT_DIR" ]; then \
				echo "Warning: Certificate directory not found at $$VESPA_CERT_DIR, trying alternatives..."; \
				VESPA_CERT_DIR=""; \
			fi; \
		fi; \
		if [ -z "$$VESPA_CERT_DIR" ] || [ ! -d "$$VESPA_CERT_DIR" ]; then \
			echo "Trying to detect from directory name..."; \
			DIR_NAME=$$(basename $(CURRENT_DIR)); \
			echo "Directory name: $$DIR_NAME"; \
			VESPA_CERT_DIR=$$(find $$HOME/.vespa -type d -name "*$$DIR_NAME*" 2>/dev/null | head -1); \
			if [ -z "$$VESPA_CERT_DIR" ]; then \
				echo "Trying to find any certificate directory..."; \
				VESPA_CERT_DIR=$$(find $$HOME/.vespa -name "data-plane-public-cert.pem" -type f 2>/dev/null | head -1 | xargs dirname 2>/dev/null || echo ""); \
			fi; \
		fi; \
	fi; \
	if [ -z "$$VESPA_CERT_DIR" ] || [ ! -d "$$VESPA_CERT_DIR" ]; then \
		echo "Error: Vespa certificate directory not found."; \
		echo "Expected location: $$HOME/.vespa/<tenant>.<app>.<instance>"; \
		echo ""; \
		echo "Available certificate directories:"; \
		find $$HOME/.vespa -name "data-plane-public-cert.pem" -type f 2>/dev/null | xargs dirname 2>/dev/null | head -5 || echo "  (none found)"; \
		echo ""; \
		echo "To fix:"; \
		echo "  1. Run 'vespa auth cert' to generate certificates"; \
		echo "  2. Or set VESPA_CERT_DIR environment variable:"; \
		echo "     export VESPA_CERT_DIR=$$HOME/.vespa/<tenant>.<app>.<instance>"; \
		exit 1; \
	fi; \
	if [ ! -f "$$VESPA_CERT_DIR/data-plane-public-cert.pem" ] || [ ! -f "$$VESPA_CERT_DIR/data-plane-private-key.pem" ]; then \
		echo "Error: Certificate files not found in $$VESPA_CERT_DIR"; \
		echo "Expected files: data-plane-public-cert.pem and data-plane-private-key.pem"; \
		exit 1; \
	fi; \
	echo "Using certificate directory: $$VESPA_CERT_DIR"; \
	docker run --rm -it \
		-v $(CURRENT_DIR)/dataset/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro \
		-v $(CURRENT_DIR)/dataset:/data:ro \
		-v $$VESPA_CERT_DIR:/certs:ro \
		--network host \
		logstash-vespa:latest