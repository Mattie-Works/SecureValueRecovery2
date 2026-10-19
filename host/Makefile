# Copyright 2024 Signal Messenger, LLC
# SPDX-License-Identifier: AGPL-3.0-only
GO_TEST_FLAGS ?=

all: build test cmds miniredis
.PHONY: all clean protos build test validate enclave_binaries

build: generated
	go build main.go

enclave_binaries:
	$(MAKE) -C ../enclave build/enclave.nsm

# -count=1 forces this test to run un-cached, since enclave.nsm may have changed
# even though Go code has not, and tests may depend on it.
test: build enclave_binaries rustclient/target/debug/rustclient
	go test $(GO_TEST_FLAGS) -count=1 ./...

rustclient/target/debug/rustclient:
	cd rustclient && cargo build

PROTO_FILES = \
  $(patsubst ../shared/proto/%.proto,proto/%.pb.go,$(wildcard ../shared/proto/*.proto)) \
  $(patsubst proto/%.proto,proto/%.pb.go,$(wildcard proto/*.proto)) \
## PROTO_FILES
# This $(firstword) trick allows for grouped targets.
$(filter-out $(firstword $(PROTO_FILES)),$(PROTO_FILES)): $(firstword $(PROTO_FILES))
$(firstword $(PROTO_FILES)): ../shared/proto/*.proto proto/*.proto
	protoc --go_out=. --go_opt=module=github.com/signalapp/svr2 --proto_path=../shared/proto ../shared/proto/*.proto
	protoc --go_out=. --go_opt=module=github.com/signalapp/svr2 --proto_path=../shared/proto --proto_path=proto proto/*.proto
protos: $(PROTO_FILES)
generated: protos

validate: generated
	go vet ./...
	CHANGES="$$(go fmt ./...)" ; echo "Changes: $$CHANGES" && [ -z "$$CHANGES" ]
 
clean:
	rm -vfr enclave/c
	rm -vfr main
	rm -vf  proto/*.pb.go

miniredis:
	(cd miniredis && go build miniredis.go)

cmds: generated
	(cd cmd/control && go build)
	(cd cmd/svr2client && go build)
	(cd cmd/svr3client && go build)
