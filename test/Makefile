default: run-tests

WASM_OPT   := ../build/binaryen/bin/wasm-opt
WASI_SDK_PATH := ../deps/wasi-sdk-25.0
WASI_SYSROOT  := $(abspath ${WASI_SDK_PATH}/share/wasi-sysroot)
OUTPUT_DIR := ../build/test

# https://github.com/caspervonb/wasi-test-suite
WASI_TEST_SUITE_SRC := ../deps/wasi-test-suite
WASI_TEST_SUITE_DST := $(OUTPUT_DIR)/wasi-test-suite
WASI_TEST_SUITE_SRC_TESTS := $(shell ls $(WASI_TEST_SUITE_SRC)/**/*.wasm)
WASI_TEST_SUITE_DST_TESTS := $(WASI_TEST_SUITE_SRC_TESTS:$(WASI_TEST_SUITE_SRC)/%.wasm=$(WASI_TEST_SUITE_DST)/%.wasm)

$(WASI_TEST_SUITE_DST)/%.wasm: $(WASI_TEST_SUITE_SRC)/%.wasm $(WASM_OPT)
	mkdir -p $(@D)
	$(WASM_OPT) -g -O --asyncify $< -o $@

# https://github.com/bytecodealliance/wasmtime/tree/main/crates/test-programs/wasi-tests
WASI_TESTS_CRATE_PATH := ../deps/wasmtime/crates/test-programs/wasi-tests
WASMTIME_BIN := $(WASI_TESTS_CRATE_PATH)/target/wasm32-wasip1/debug
WASMTIME_SRC := $(WASI_TESTS_CRATE_PATH)/src/bin
WASMTIME_DST := $(OUTPUT_DIR)/wasmtime
WASMTIME_SRC_TESTS := $(shell ls $(WASI_TESTS_CRATE_PATH)/src/bin/*.rs)
WASMTIME_DST_TESTS := $(WASMTIME_SRC_TESTS:$(WASMTIME_SRC)/%.rs=$(WASMTIME_DST)/%.wasm)

BENCHMARK_SRC := ./subjects
BENCHMARK_DST := $(OUTPUT_DIR)/benchmark
BENCHMARK_SRC_TESTS := $(shell ls ./subjects/*.c)
BENCHMARK_DST_TESTS := $(BENCHMARK_SRC_TESTS:$(BENCHMARK_SRC)/%.c=$(BENCHMARK_DST)/%.wasm)

$(WASMTIME_BIN)/%.wasm: $(wildcard $(WASI_TESTS_CRATE_PATH)/src/**)
	cargo build --bin $* --target wasm32-wasip1 --manifest-path $(WASI_TESTS_CRATE_PATH)/Cargo.toml

$(WASMTIME_DST)/%.wasm: $(WASMTIME_BIN)/%.wasm $(WASM_OPT)
	mkdir -p $(@D)
	$(WASM_OPT) -g -O --asyncify $< -o $@

BUNDLE := $(OUTPUT_DIR)/index.mjs
MEMFS_SRC  := ../dist/memfs.wasm
MEMFS_DST  := $(OUTPUT_DIR)/memfs.wasm

node_modules: ./package.json ./package-lock.json ../package.json
	npm install --no-audit --no-fund --no-progress --quiet
	touch $@

$(OUTPUT_DIR)/standalone.mjs: $(BENCHMARK_DST_TESTS) node_modules ./driver/*.ts
	node benchmark-build.mjs

run-tests: $(BUNDLE) $(OUTPUT_DIR)/wasm-table.ts node_modules $(OUTPUT_DIR)/standalone.mjs
	$(shell npm root)/.bin/tsc -p ./tsconfig.json
	JEST_JUNIT_OUTPUT_DIR=$(OUTPUT_DIR) OUTPUT_DIR=$(OUTPUT_DIR) NODE_OPTIONS=--experimental-vm-modules NODE_NO_WARNINGS=1 \
        $(shell npm root)/.bin/jest --detectOpenHandles -i

$(OUTPUT_DIR)/wasm-table.ts: $(WASI_TEST_SUITE_DST_TESTS) $(WASMTIME_DST_TESTS)
	mkdir -p $(@D)
	node ./generate-wasm-table.mjs $(OUTPUT_DIR) > $@

$(MEMFS_DST): $(MEMFS_SRC)
	mkdir -p $(@D)
	cp $< $@

$(BUNDLE): $(wildcard ../dist/**) $(wildcard ./driver/**) $(MEMFS_DST) $(OUTPUT_DIR)/wasm-table.ts node_modules
	mkdir -p $(@D)
	$(shell npm root)/.bin/esbuild --bundle ./driver/worker.ts --outfile=$@ --format=esm --log-level=warning --external:*.wasm

$(WASM_OPT):
	@$(call color,"downloading binaryen")
	mkdir -p $(@D)
	curl -Lo binaryen.tar.gz https://github.com/WebAssembly/binaryen/releases/download/version_100/binaryen-version_100-x86_64-linux.tar.gz
	echo '9057c8f3f0bbfec47a95985c8f0faad8cc2aa3932e94a7d6b705e245ed140e19  binaryen.tar.gz' | sha256sum -c
	tar zxvf binaryen.tar.gz --strip-components=1 --touch -C ../build/binaryen
	rm binaryen.tar.gz

export WASI_CC      := $(abspath ${WASI_SDK_PATH}/bin/clang) -target wasm32-wasip1 --sysroot=${WASI_SYSROOT}
export WASI_CFLAGS  := -Oz -flto
export WASI_LDFLAGS := -flto -Wl,--allow-undefined

$(BENCHMARK_DST)/%.wasm: $(WASI_SDK_PATH) $(WASM_OPT) $(BENCHMARK_SRC)/%.c
	mkdir -p $(BENCHMARK_DST)
	$(WASI_CC) $(WASI_CFLAGS) $(WASI_LDFLAGS) subjects/$*.c -o $(BENCHMARK_DST)/$*.wasm 
	$(WASM_OPT) -g -O --asyncify $(BENCHMARK_DST)/$*.wasm -o $(BENCHMARK_DST)/$*.asyncify.wasm
