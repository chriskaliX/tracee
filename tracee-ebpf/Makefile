.PHONY: all
all: build bpf

# tools:
CMD_LLC ?= llc
CMD_CLANG ?= clang
CMD_LLVM_STRIP ?= llvm-strip
CMD_DOCKER ?= docker
CMD_GIT ?= git
# environment:
ARCH_UNAME := $(shell uname -m)
ARCH ?= $(ARCH_UNAME:aarch64=arm64)
KERN_RELEASE ?= $(shell uname -r)
KERN_BLD_PATH ?= $(if $(KERN_HEADERS),$(KERN_HEADERS),/lib/modules/$(KERN_RELEASE)/build)
KERN_SRC_PATH ?= $(if $(KERN_HEADERS),$(KERN_HEADERS),$(if $(wildcard /lib/modules/$(KERN_RELEASE)/source),/lib/modules/$(KERN_RELEASE)/source,$(KERN_BLD_PATH)))
LAST_GIT_TAG ?= $(shell $(CMD_GIT) describe --tags 2>/dev/null)
VERSION ?= $(if $(RELEASE_TAG),$(RELEASE_TAG),$(notdir $(LAST_GIT_TAG)))
# inputs and outputs:
OUT_DIR ?= dist
GO_SRC := $(shell find . -type f -name '*.go')
OUT_BIN := $(OUT_DIR)/tracee-ebpf
BPF_SRC := tracee/tracee.bpf.c
OUT_BPF := $(OUT_DIR)/tracee.bpf.$(subst .,_,$(KERN_RELEASE)).$(subst .,_,$(VERSION)).o
OUT_BPF_CORE := $(OUT_DIR)/tracee.bpf.core.o
BPF_HEADERS := ../3rdparty/include
BPF_BUNDLE := $(OUT_DIR)/tracee.bpf.tar.gz
LIBBPF_SRC := ../3rdparty/libbpf/src
LIBBPF_HEADERS := $(OUT_DIR)/libbpf/usr/include
LIBBPF_OBJ := $(OUT_DIR)/libbpf/libbpf.a
OS_RELEASE_FILE := /etc/os-release

# static build:
ifdef STATIC
CGO_EXT_LDFLAGS += -static
endif
OUT_DOCKER ?= tracee
DOCKER_BUILDER ?= tracee-builder
# DOCKER_BUILDER_KERN_SRC(/BLD) is where the docker builder looks for kernel headers
DOCKER_BUILDER_KERN_BLD ?= $(if $(shell readlink -f $(KERN_BLD_PATH)),$(shell readlink -f $(KERN_BLD_PATH)),$(KERN_BLD_PATH))
DOCKER_BUILDER_KERN_SRC ?= $(if $(shell readlink -f $(KERN_SRC_PATH)),$(shell readlink -f $(KERN_SRC_PATH)),$(KERN_SRC_PATH))
# DOCKER_BUILDER_KERN_SRC_MNT is the kernel headers directory to mount into the docker builder container. DOCKER_BUILDER_KERN_SRC should usually be a descendent of this path.
DOCKER_BUILDER_KERN_SRC_MNT ?= $(dir $(DOCKER_BUILDER_KERN_SRC))
LINUX_VERSION_CODE := $(shell uname -r | awk '{split($$0,a,"."); split(a[3], b, "-"); print lshift(a[1], 16) + lshift(a[2],8) + b[1];}')

$(OUT_DIR):
	mkdir -p $@

.PHONY: build
build: $(OUT_BIN)

go_env := GOOS=linux GOARCH=$(ARCH:x86_64=amd64) CC=$(CMD_CLANG) CGO_CFLAGS="-I $(abspath $(LIBBPF_HEADERS))" CGO_LDFLAGS="$(abspath $(LIBBPF_OBJ))"
ifndef DOCKER
$(OUT_BIN): $(LIBBPF_HEADERS) $(LIBBPF_OBJ) $(OUT_BPF_CORE) $(filter-out *_test.go,$(GO_SRC)) $(BPF_BUNDLE) | $(OUT_DIR)
	# workaround until Makefile.one becomes standard
	mkdir -p ../dist/
	mkdir -p ../dist/btfhub
	touch ../dist/btfhub/.place-holder
	cp -rfp ./dist/* ../dist/
	# end of workaround
	$(go_env) go build \
		-tags ebpf,netgo \
		-ldflags="\
			-w -extldflags \"$(CGO_EXT_LDFLAGS)\" \
			-X main.version=\"$(VERSION)\" \
			" \
		-v -o ./dist/ \
		../cmd/tracee-ebpf
else
$(OUT_BIN): $(DOCKER_BUILDER) | $(OUT_DIR)
	$(call docker_builder_make,$@ VERSION=$(VERSION))
endif

bpf_compile_tools = $(CMD_LLC) $(CMD_CLANG)
.PHONY: $(bpf_compile_tools)
$(bpf_compile_tools): % : check_%

$(LIBBPF_SRC):
	test -d $(LIBBPF_SRC) || (echo "missing libbpf source - maybe do 'git submodule init && git submodule update'" ; false)

$(LIBBPF_HEADERS) $(LIBBPF_HEADERS)/bpf $(LIBBPF_HEADERS)/linux: | $(OUT_DIR) $(bpf_compile_tools) $(LIBBPF_SRC)
	cd $(LIBBPF_SRC) && $(MAKE) install_headers install_uapi_headers DESTDIR=$(abspath $(OUT_DIR))/libbpf

$(LIBBPF_OBJ): | $(OUT_DIR) $(bpf_compile_tools) $(LIBBPF_SRC)
	cd $(LIBBPF_SRC) && $(MAKE) OBJDIR=$(abspath $(OUT_DIR))/libbpf BUILD_STATIC_ONLY=1

bpf_bundle_dir := $(OUT_DIR)/tracee.bpf
$(BPF_BUNDLE): $(BPF_SRC) $(LIBBPF_HEADERS)/bpf $(BPF_HEADERS)
	mkdir -p $(bpf_bundle_dir)
	cp $$(find $^ -type f) $(bpf_bundle_dir)
	# workaround until Makefile.one becomes standard
	mkdir -p ../dist/
	cp -rf $(bpf_bundle_dir) ../dist/
	# end of workaround

.PHONY: bpf
bpf: $(OUT_BPF) $(OUT_BPF_CORE)

linux_arch := $(ARCH:x86_64=x86)
ifndef DOCKER
$(OUT_DIR)/tracee.bpf.%.o: $(BPF_SRC) $(LIBBPF_HEADERS) | $(OUT_DIR) $(bpf_compile_tools)
	@v=$$($(CMD_CLANG) --version); test $$(echo $${v#*version} | head -n1 | cut -d '.' -f1) -ge '9' || (echo 'required minimum clang version: 9' ; false)
	$(CMD_CLANG) -S \
		-D__BPF_TRACING__ \
		-D__KERNEL__ \
		-D__TARGET_ARCH_$(linux_arch) \
		-I $(LIBBPF_HEADERS)/bpf \
		-include $(KERN_SRC_PATH)/include/linux/kconfig.h \
		-I $(KERN_SRC_PATH)/arch/$(linux_arch)/include \
		-I $(KERN_SRC_PATH)/arch/$(linux_arch)/include/uapi \
		-I $(KERN_BLD_PATH)/arch/$(linux_arch)/include/generated \
		-I $(KERN_BLD_PATH)/arch/$(linux_arch)/include/generated/uapi \
		-I $(KERN_SRC_PATH)/include \
		-I $(KERN_BLD_PATH)/include \
		-I $(KERN_SRC_PATH)/include/uapi \
		-I $(KERN_BLD_PATH)/include/generated \
		-I $(KERN_BLD_PATH)/include/generated/uapi \
		-I $(BPF_HEADERS) \
		-Wunused \
		-Wall \
		-Wno-frame-address \
		-Wno-unused-value \
		-Wno-unknown-warning-option \
		-Wno-pragma-once-outside-header \
		-Wno-pointer-sign \
		-Wno-gnu-variable-sized-type-not-at-end \
		-Wno-deprecated-declarations \
		-Wno-compare-distinct-pointer-types \
		-Wno-address-of-packed-member \
		-fno-stack-protector \
		-fno-jump-tables \
		-fno-unwind-tables \
		-fno-asynchronous-unwind-tables \
		-xc \
		-nostdinc \
		-O2 -emit-llvm -c -g $< -o $(@:.o=.ll)
	$(CMD_LLC) -march=bpf -filetype=obj -o $@ $(@:.o=.ll)
	-$(CMD_LLVM_STRIP) -g $@
	rm $(@:.o=.ll)

$(OUT_BPF_CORE): $(BPF_SRC) $(LIBBPF_HEADERS) $(CMD_CLANG)
	$(CMD_CLANG) \
		-I $(LIBBPF_HEADERS)/bpf \
		-I $(BPF_HEADERS) \
		-I tracee/co-re \
		-DCORE \
		-D__TARGET_ARCH_$(linux_arch) \
		-D__BPF_TRACING__ \
		-D__KERNEL__ \
		-target bpf \
		-O2 -c -g -o $@ $<
else
#docker yes
$(OUT_BPF): $(DOCKER_BUILDER) | $(OUT_DIR)
	$(call docker_builder_make,$@)

$(OUT_BPF_CORE): $(DOCKER_BUILDER) | $(OUT_DIR)
	$(call docker_builder_make,$@)
endif

.PHONY: test
ifndef DOCKER
test: $(GO_SRC) $(LIBBPF_HEADERS) $(LIBBPF_OBJ)
	$(go_env) go test \
		-tags ebpf $(GOTEST_FLAGS) \
		-v \
		../cmd/tracee-ebpf/ \
		./...

else
test: $(DOCKER_BUILDER)
	$(call docker_builder_make,$@)
endif

.PHONY: test-integration
test-integration: $(OUT_BIN)
	$(go_env) TRC_BIN=../../$(OUT_BIN) go test -exec 'sudo -E' -v test/integration/*_test.go

# record built image id to prevent unnecessary building and for cleanup
docker_builder_file := $(OUT_DIR)/$(DOCKER_BUILDER)
.PHONY: $(DOCKER_BUILDER)
$(DOCKER_BUILDER) $(docker_builder_file) &: Dockerfile.builder | $(OUT_DIR) check_$(CMD_DOCKER)
	$(CMD_DOCKER) build -t $(DOCKER_BUILDER) --iidfile $(docker_builder_file) - < $<

# docker_builder_make runs a make command in the tracee-builder container
define docker_builder_make
	$(CMD_DOCKER) run --rm \
	-v $(abspath $(DOCKER_BUILDER_KERN_SRC_MNT)):$(DOCKER_BUILDER_KERN_SRC_MNT) \
	-v $(abspath ./..):/tracee \
	-w /tracee/tracee-ebpf \
	--entrypoint make $(DOCKER_BUILDER) KERN_BLD_PATH=$(DOCKER_BUILDER_KERN_BLD) KERN_SRC_PATH=$(DOCKER_BUILDER_KERN_SRC) $(1)
endef

.PHONY: mostlyclean
mostlyclean:
	-rm -rf $(OUT_BIN) $(bpf_bundle_dir) $(OUT_BPF) $(OUT_BPF_CORE) $(BPF_BUNDLE)

.PHONY: clean
clean:
	-FILE="$(docker_builder_file)" ; \
	if [ -r "$$FILE" ] ; then \
		$(CMD_DOCKER) rmi "$$(< $$FILE)" ; \
	fi
	-rm -rf $(OUT_DIR)
	$(MAKE) -C $(LIBBPF_SRC) clean

check_%:
	@command -v $* >/dev/null || (echo "missing required tool $*" ; false)

.PHONY: docker
docker:
	$(CMD_DOCKER) build --build-arg VERSION=$(VERSION) -t $(OUT_DOCKER):latest .

.PHONY: docker-slim
docker-slim:
	$(CMD_DOCKER) build --build-arg VERSION=$(VERSION) -t $(OUT_DOCKER):slim --build-arg BASE=slim .
