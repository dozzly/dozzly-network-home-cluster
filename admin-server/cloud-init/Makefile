# =========================
# Makefile — admin-server (.env-driven, no SOPS)
# Usage:
#   # Put variables in ./.env (same dir as this Makefile), e.g.:
#   #   admin_user=alice
#   #   ssh_pubkey=ssh-ed25519 AAAA... alice@example
#   #   ssh_password_hash=$y$j9T$...
#   #   admin_server_ip=192.168.1.50
#   #   admin_gateway=192.168.1.1
#   #   admin_dns=1.1.1.1
#   make clean         # if you have a clean target
#   set -a; . ./.env; set +a   # export the env into your shell
#   make render
#   make seed
#   make run
# =========================

SHELL := /bin/bash
.ONESHELL:

# ---- paths & image config ----
IMG_DIR          ?= ..
IMAGE           := $(abspath $(strip $(IMG_DIR)/noble-server-cloudimg-arm64.img))
DISK            := $(abspath $(strip $(IMG_DIR)/admin-server.qcow2))
SEED            := $(abspath $(strip seed.iso))
DISK_SIZE       ?= 80G
INSTANCE_NAME   ?= admin-server
VM_HOSTNAME     ?= atherton
UEFI            ?= /opt/homebrew/share/qemu/edk2-aarch64-code.fd

# ---- networking (bridged) ----
BRIDGE_IF       ?= en0
VM_MAC          ?= 52:54:00:a1:b2:c3

# ---- .env wiring ----
ENV_FILE ?= .env

# Robust loader (pure bash):
# - Accepts KEY=VALUE and "key: value"
# - Strips comments and blank lines
# - Trims whitespace, optionally removes surrounding single/double quotes
# - Exports each variable safely via export "KEY=VALUE" (no eval)
define DOTENV
if [ -f "$(ENV_FILE)" ]; then \
	set -a; \
	if grep -Eq '^[[:space:]]*[A-Za-z_][A-Za-z0-9_]*[[:space:]]*:' "$(ENV_FILE)"; then \
		echo "$(ENV_FILE) appears to contain YAML-style lines (with a colon). Convert to KEY=VALUE."; \
		exit 1; \
	fi; \
	. "$(ENV_FILE)"; \
	set +a; \
fi
endef
export DOTENV

.PHONY: debug-dotenv
debug-dotenv:
	@echo "== DOTENV debug =="
	@echo "PWD: $$(pwd)"
	@echo "ENV_FILE: $(ENV_FILE)"
	@echo "--- .env (first 20 lines) ---"
	@{ test -f "$(ENV_FILE)" && sed -n '1,20p' "$(ENV_FILE)"; } || echo "(missing .env)"
	@echo "----------- source via DOTENV -----------"
	@set -x
	@$(DOTENV)
	@set +x
	@echo "----------- after source ---------------"
	@(declare -p admin_user ssh_pubkey ssh_password_hash admin_server_ip admin_gateway admin_dns 2>/dev/null || true)

# Required vars for core templates/usage
REQUIRED_SECRETS := admin_user ssh_pubkey ssh_password_hash

# Helper: error if tool missing
req = if ! command -v $(1) >/dev/null 2>&1; then echo "Missing dependency: $(1)"; exit 1; fi


.PHONY: help
help:
	@printf "Targets:\n"
	@printf "  render         render cloud-init files (user/meta[/network]) from .env\n"
	@printf "  seed           build cloud-init seed ISO (CIDATA)\n"
	@printf "  prepare-disk   create qcow2 overlay over base cloud image\n"
	@printf "  run            boot VM (bridged)\n"
	@printf "  imginfo        show base image info\n"
	@printf "  print-secrets-env  list vars loaded from .env (masked)\n"
	@printf "  env-check      fail if .env is missing\n"
	@printf "  clean          remove rendered/ISO files\n"

.DEFAULT_GOAL := help

# ---- diagnostics ----
.PHONY: print-secrets-env env-check imginfo secrets-ok

env-check:
	@test -f "$(ENV_FILE)" || { echo "Missing $(ENV_FILE). Create it next to this Makefile."; exit 1; }

.PHONY: secrets-ok
secrets-ok:
	@$(DOTENV)
	@missing=0; \
	for v in admin_user ssh_pubkey ssh_password_hash; do \
	  val=$$(eval echo "\$$$$v"); \
	  if [ -z "$$val" ]; then echo "Missing $$v in environment (.env)."; missing=1; fi; \
	done; \
	[ "$$missing" -eq 0 ] && echo "secrets: OK" || exit 1

print-secrets-env:
	@$(DOTENV)
	@echo "Loaded from $(ENV_FILE):"
	@for k in $(REQUIRED_SECRETS) admin_server_ip admin_gateway admin_dns; do \
	  v="$${!k:-}"; \
	  if [ -n "$$v" ]; then printf "  %s=%s\n" "$$k" "******"; else printf "  %s=<unset>\n" "$$k"; fi; \
	done

imginfo:
	@$(call req,qemu-img)
	@qemu-img info '$(IMAGE)'

# ---- render cloud-init ----
meta-data.tmpl:
	@echo "instance-id: $$INSTANCE_NAME" > $@
	@echo "local-hostname: $$VM_HOSTNAME" >> $@
	@echo "created minimal $@"

user-data: user-data.tmpl
	@$(DOTENV)
	@missing=0; \
	for v in admin_user ssh_pubkey ssh_password_hash; do \
	  val=$$(eval echo "\$$$$v"); \
	  if [ -z "$$val" ]; then echo "Missing $$v in environment (.env)."; missing=1; fi; \
	done; \
	[ "$$missing" -eq 0 ] || exit 1
	@envsubst "\$$admin_user \$$ssh_pubkey \$$ssh_password_hash \$$VM_HOSTNAME" < $< > $@
	@echo "wrote $@"
	
# Optional network-config: substitutes whatever vars are present
network-config: network-config.tmpl
	@$(DOTENV)
	@envsubst "\$$admin_server_ip \$$admin_gateway \$$admin_dns" < $< > $@
	@echo "wrote $@"

# ---- render cloud-init ----
meta-data.tmpl:
	@printf 'instance-id: %s\nlocal-hostname: %s\n' "$(INSTANCE_NAME)" "$(VM_HOSTNAME)" > $@
	@echo "created $@"

# If you also need the non-template file:
meta-data: meta-data.tmpl
	@cp -f $< $@

render: user-data meta-data $(if $(wildcard network-config.tmpl),network-config)
	@echo "Rendered cloud-init files."

# ---- seed ISO ----
seed: render
	@$(call req,hdiutil)
	@tmpdir="$$(mktemp -d -t cidata.XXXXXX)"; \
	trap 'rm -rf "$$tmpdir"' EXIT INT TERM; \
	cp user-data "$$tmpdir/user-data"; \
	cp meta-data "$$tmpdir/meta-data"; \
	if [ -f network-config ]; then cp network-config "$$tmpdir/network-config"; fi; \
	rm -f '$(SEED)'; \
	hdiutil makehybrid -o '$(SEED)' -iso -joliet -default-volume-name CIDATA "$$tmpdir" >/dev/null; \
	echo "wrote $(SEED)"

# ---- backing image / overlay disk ----
prepare-disk:
	@$(call req,qemu-img)
	@test -f '$(DISK)' || qemu-img create -f qcow2 -F qcow2 -b '$(IMAGE)' '$(DISK)'
	@qemu-img resize '$(DISK)' $(DISK_SIZE)
	@echo "overlay: $(DISK)"

# ---- run VM ----
run: seed prepare-disk
	@$(call req,qemu-system-aarch64)
	@qemu-system-aarch64 \
		-name $(INSTANCE_NAME) \
		-machine virt,highmem=on,accel=hvf \
		-cpu host -smp 2 -m 2G \
		-drive if=pflash,format=raw,unit=0,readonly=on,file=/opt/homebrew/share/qemu/edk2-aarch64-code.fd \
		-drive if=pflash,format=raw,unit=1,file=./edk2-vars.aarch64.fd \
		-device virtio-scsi-pci,id=scsi0 \
		-drive if=none,id=vdisk,file='$(DISK)',format=qcow2,cache=none,discard=unmap \
		-device scsi-hd,drive=vdisk,bus=scsi0.0,bootindex=1 \
		-drive if=none,id=cidata,media=cdrom,file='$(SEED)',format=raw \
		-device scsi-cd,drive=cidata,bus=scsi0.0,bootindex=2 \
		-nic vmnet-bridged,model=virtio-net-pci,ifname=$(BRIDGE_IF),mac=$(VM_MAC) \
		-boot order=c,menu=on \
		-serial mon:stdio -nographic

# ---- clean ----
clean:
	rm -f user-data meta-data network-config $(SEED)
