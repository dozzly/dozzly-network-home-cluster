SHELL := /bin/sh
-include .env

VM_NAME ?= warner
CPUS ?= 2
RAM ?= 4096
DISK ?= $(VM_NAME).qcow2
DISK_SIZE ?= 40G
MAC ?= AA:BB:CC:DD:EE:03
BRIDGE_IF ?= en0
EFI_VARS ?= cloud-init/edk2-vars.aarch64.fd
QEMU ?= qemu-system-aarch64
SUDO ?= sudo

# Try to auto-locate edk2 firmware (POSIX sh so it works everywhere)
EFI_CODE ?= $(shell \
  if [ -f /opt/homebrew/share/qemu/edk2-aarch64-code.fd ]; then \
    echo /opt/homebrew/share/qemu/edk2-aarch64-code.fd; \
  elif [ -f /usr/local/share/qemu/edk2-aarch64-code.fd ]; then \
    echo /usr/local/share/qemu/edk2-aarch64-code.fd; \
  elif [ -f /usr/share/qemu/edk2-aarch64-code.fd ]; then \
    echo /usr/share/qemu/edk2-aarch64-code.fd; \
  fi)

VARS_SRC ?= $(shell \
  if [ -f /opt/homebrew/share/qemu/edk2-aarch64-vars.fd ]; then \
    echo /opt/homebrew/share/qemu/edk2-aarch64-vars.fd; \
  elif [ -f /usr/local/share/qemu/edk2-aarch64-vars.fd ]; then \
    echo /usr/local/share/qemu/edk2-aarch64-vars.fd; \
  elif [ -f /usr/share/qemu/edk2-aarch64-vars.fd ]; then \
    echo /usr/share/qemu/edk2-aarch64-vars.fd; \
  fi)

.PHONY: help check vars disk run run-gui clean

help:
	@echo "Targets:"
	@echo "  make run       # create disk (if missing) and PXE boot aarch64 VM"
	@echo "  make run-gui   # same, but show a simple window"
	@echo "  make disk      # create $(DISK) ($(DISK_SIZE))"
	@echo "  make vars      # copy a fresh UEFI VARS file to $(EFI_VARS)"
	@echo "  make clean     # remove disk image"

check:
	@if ! command -v $(QEMU) >/dev/null 2>&1; then \
	  echo "qemu-system-aarch64 not found. Install with: brew install qemu" >&2; exit 1; fi
	@if [ -z "$(EFI_CODE)" ] || [ ! -f "$(EFI_CODE)" ]; then \
	  echo "EDK2 firmware (edk2-aarch64-code.fd) not found. Try: brew install qemu" >&2; exit 1; fi
	@if [ ! -f "$(EFI_VARS)" ]; then \
	  echo "$(EFI_VARS) missing. Run: make vars" >&2; exit 1; fi

vars:
	@if [ -z "$(VARS_SRC)" ] || [ ! -f "$(VARS_SRC)" ]; then \
	  echo "Could not locate edk2-aarch64-vars.fd. Try: brew install qemu" >&2; exit 1; fi
	mkdir -p "$$(dirname "$(EFI_VARS)")"
	cp "$(VARS_SRC)" "$(EFI_VARS)"

$(DISK):
	qemu-img create -f qcow2 "$(DISK)" "$(DISK_SIZE)"

disk: $(DISK)

run: check disk
	@echo "Booting $(VM_NAME) (bridge $(BRIDGE_IF), MAC $(MAC))"
	$(SUDO) $(QEMU) \
	  -machine virt,accel=hvf,highmem=on \
	  -cpu host \
	  -smp $(CPUS) -m $(RAM) \
	  -name "$(VM_NAME)" \
	  -drive if=pflash,format=raw,readonly=on,file="$(EFI_CODE)" \
	  -drive if=pflash,format=raw,file="$(EFI_VARS)" \
	  -netdev vmnet-bridged,id=net0,ifname=$(BRIDGE_IF) \
	  -device virtio-net-pci,netdev=net0,mac=$(MAC),bootindex=1 \
	  -drive if=none,id=vdisk0,file="$(DISK)",format=qcow2,cache=none,aio=threads \
	  -device virtio-blk-pci,drive=vdisk0 \
	  -boot order=n,menu=on \
	  -serial mon:stdio

run-gui: check disk
	@echo "Booting $(VM_NAME) (GUI) (bridge $(BRIDGE_IF), MAC $(MAC))"
	$(SUDO) $(QEMU) \
	  -machine virt,accel=hvf,highmem=on \
	  -cpu host \
	  -smp $(CPUS) -m $(RAM) \
	  -name "$(VM_NAME)" \
	  -drive if=pflash,format=raw,readonly=on,file="$(EFI_CODE)" \
	  -drive if=pflash,format=raw,file="$(EFI_VARS)" \
	  -netdev vmnet-bridged,id=net0,ifname=$(BRIDGE_IF) \
	  -device virtio-net-pci,netdev=net0,mac=$(MAC),bootindex=1 \
	  -drive if=none,id=vdisk0,file="$(DISK)",format=qcow2,cache=none,aio=threads \
	  -device virtio-blk-pci,drive=vdisk0 \
	  -boot order=n \
	  -serial mon:stdio \
	  -display default,show-cursor=on

clean:
	rm -f "$(DISK)"
