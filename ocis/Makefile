SHELL := bash
NAME := ocis
IMPORT := github.com/owncloud/ocis/$(NAME)
BIN := bin
DIST := dist
HUGO := hugo
CONFIG := config/identifier-registration.yaml

ifeq ($(OS), Windows_NT)
	EXECUTABLE := $(NAME).exe
	UNAME := Windows
else
	EXECUTABLE := $(NAME)
	UNAME := $(shell uname -s)
endif

ifeq ($(UNAME), Darwin)
	GOBUILD ?= go build -i
else
	GOBUILD ?= go build
endif

PACKAGES ?= $(shell go list ./...)
SOURCES ?= $(shell find . -name "*.go" -type f)
GENERATE ?= $(PACKAGES)

TAGS ?=

ifndef OUTPUT
	ifneq ($(DRONE_TAG),)
		OUTPUT ?= $(subst v,,$(DRONE_TAG))
	else
		OUTPUT ?= testing
	endif
endif

ifndef VERSION
	ifneq ($(DRONE_TAG),)
		VERSION ?= $(subst v,,$(DRONE_TAG))
	else
		VERSION ?= $(shell git rev-parse --short HEAD)
	endif
endif

ifndef DATE
	DATE := $(shell date -u '+%Y%m%d')
endif

LDFLAGS += -s -w -X "$(IMPORT)/pkg/version.String=$(VERSION)" -X "$(IMPORT)/pkg/version.Date=$(DATE)"
DEBUG_LDFLAGS += -X "$(IMPORT)/pkg/version.String=$(VERSION)" -X "$(IMPORT)/pkg/version.Date=$(DATE)"
GCFLAGS += all=-N -l

.PHONY: all
all: build

.PHONY: sync
sync:
	go mod download

.PHONY: clean
clean: clean-config
	go clean -i ./...
	rm -rf $(BIN) $(DIST) $(HUGO)

.PHONY: clean-config
clean-config:
	rm -rf $(CONFIG)

.PHONY: fmt
fmt:
	gofmt -s -w $(SOURCES)

.PHONY: vet
vet:
	go vet $(PACKAGES)

.PHONY: lint
lint:
	for PKG in $(PACKAGES); do go run golang.org/x/lint/golint -set_exit_status $$PKG || exit 1; done;

.PHONY: generate
generate:
	go generate $(GENERATE)

.PHONY: changelog
changelog:
	go run github.com/restic/calens -i ../changelog -t ../changelog/CHANGELOG.tmpl >| ../CHANGELOG.md

.PHONY: test
test:
	go run github.com/haya14busa/goverage -v -coverprofile coverage.out $(PACKAGES)

.PHONY: install
install: $(SOURCES)
	go install -v -tags '$(TAGS)' -ldflags '$(LDFLAGS)' ./cmd/$(NAME)

.PHONY: build
build: $(BIN)/$(EXECUTABLE) $(BIN)/$(EXECUTABLE)-debug

$(BIN)/$(EXECUTABLE): $(SOURCES)
	$(GOBUILD) -v -tags '$(TAGS)' -ldflags '$(LDFLAGS)' -o $@ ./cmd/$(NAME)

$(BIN)/$(EXECUTABLE)-debug: $(SOURCES)
	$(GOBUILD) -v -tags '$(TAGS)' -ldflags '$(DEBUG_LDFLAGS)' -gcflags '$(GCFLAGS)' -o $@ ./cmd/$(NAME)

$(BIN)/$(EXECUTABLE)-linux: $(SOURCES)
	GOOS=linux GOARCH=amd64 $(GOBUILD) -v -tags '$(TAGS)' -ldflags '$(LDFLAGS)' -gcflags '$(GCFLAGS)' -o $@ ./cmd/$(NAME)

.PHONY: staticcheck
staticcheck:
	go run honnef.co/go/tools/cmd/staticcheck -tags '$(TAGS)' $(PACKAGES)

.PHONY: release
release: release-dirs release-linux release-windows release-darwin release-copy release-check

.PHONY: release-dirs
release-dirs:
	mkdir -p $(DIST)/binaries $(DIST)/release

.PHONY: release-linux
release-linux: release-dirs
	go run github.com/mitchellh/gox -tags 'netgo $(TAGS)' -ldflags '-extldflags "-static" $(LDFLAGS)' -os 'linux' -arch 'amd64 386 arm64 arm' -output '$(DIST)/binaries/$(EXECUTABLE)-$(OUTPUT)-{{.OS}}-{{.Arch}}' ./cmd/$(NAME)

.PHONY: release-windows
release-windows: release-dirs
	go run github.com/mitchellh/gox -tags 'netgo $(TAGS)' -ldflags '-extldflags "-static" $(LDFLAGS)' -os 'windows' -arch 'amd64' -output '$(DIST)/binaries/$(EXECUTABLE)-$(OUTPUT)-{{.OS}}-{{.Arch}}' ./cmd/$(NAME)

.PHONY: release-darwin
release-darwin: release-dirs
	go run github.com/mitchellh/gox -tags 'netgo $(TAGS)' -ldflags '$(LDFLAGS)' -os 'darwin' -arch 'amd64' -output '$(DIST)/binaries/$(EXECUTABLE)-$(OUTPUT)-{{.OS}}-{{.Arch}}' ./cmd/$(NAME)

.PHONY: release-copy
release-copy:
	$(foreach file,$(wildcard $(DIST)/binaries/$(EXECUTABLE)-*),cp $(file) $(DIST)/release/$(notdir $(file));)

.PHONY: release-check
release-check:
	cd $(DIST)/release; $(foreach file,$(wildcard $(DIST)/release/$(EXECUTABLE)-*),sha256sum $(notdir $(file)) > $(notdir $(file)).sha256;)

.PHONY: release-finish
release-finish: release-copy release-check

.PHONY: config-docs-generate
config-docs-generate:
	go run github.com/owncloud/flaex >| ../docs/ocis/configuration.md

.PHONY: watch
watch:
	go run github.com/cespare/reflex -c reflex.conf

BEHAT_BIN=vendor-bin/behat/vendor/bin/behat

.PHONY: test-acceptance-api
test-acceptance-api: vendor-bin/behat/vendor
	BEHAT_BIN=$(BEHAT_BIN) $(PATH_TO_CORE)/tests/acceptance/run.sh --remote --type api

vendor/bamarni/composer-bin-plugin: composer.lock
	composer install

vendor-bin/behat/vendor: vendor/bamarni/composer-bin-plugin vendor-bin/behat/composer.lock
	composer bin behat install --no-progress

vendor-bin/behat/composer.lock: vendor-bin/behat/composer.json
	@echo behat composer.lock is not up to date.

composer.lock: composer.json
	@echo composer.lock is not up to date.