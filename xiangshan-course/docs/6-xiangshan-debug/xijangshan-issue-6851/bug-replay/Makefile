# Adapted from davidharrishmc/xiangshan-repros, xiangshan-vector-trigger-tval.
# Defaults to the upstream bare-metal toolchain; existing ELF is an archived input.
RVCC ?= riscv64-unknown-elf-gcc
RVOBJDUMP ?= riscv64-unknown-elf-objdump
OUTPUT_DIR ?= .
FLAGS = -march=rv64gcv_zicsr -mabi=lp64 -nostdlib -nostartfiles -static -T link.ld

.PHONY: all
all: $(OUTPUT_DIR)/vtrig.elf $(OUTPUT_DIR)/vtrig.dump

$(OUTPUT_DIR)/vtrig.elf: vtrig.S link.ld
	mkdir -p "$(OUTPUT_DIR)"
	$(RVCC) $(FLAGS) -o "$@" vtrig.S

$(OUTPUT_DIR)/vtrig.dump: $(OUTPUT_DIR)/vtrig.elf
	$(RVOBJDUMP) -d "$<" > "$@"
