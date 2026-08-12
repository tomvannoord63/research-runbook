# <PROJECT> — experiment driver
#
# Copied from the research-runbook template. To adopt:
#   1. Set PROJECT below.
#   2. Set the entry points (TRAIN/EVAL/PROBE) to whatever your code will be called.
#   3. Set DIAG_PREFIX to your domain diagnostic prefix (see CONVENTIONS.md).
#   4. Tune WATCH_PATTERN for your stack's crash strings.
# Everything else works unchanged.
#
# Inspection targets work before any code exists — they only parse the log conventions.
# Run targets check for their entry point and fail with a pointer rather than a traceback,
# so this file goes live unchanged the day the code lands.
#
# Usage:  make exp EXP=e1-baseline WANDB=1
#         make gates-exp EXP=e1-baseline

SHELL := /bin/bash
.DEFAULT_GOAL := help

# --- configuration -----------------------------------------------------------------
PROJECT       ?= myproject
PY            ?= uv run python
LOGDIR        ?= /tmp
LOG            = $(LOGDIR)/$(PROJECT)-$(EXP).log
WANDB         ?= 0
WANDB_PROJECT ?= $(PROJECT)

# Your domain diagnostic prefix — the mechanism the headline metric cannot observe.
DIAG_PREFIX   ?= DOMAIN

# What a Monitor watches. Keep high-volume diagnostics OUT of this: one line per layer
# per checkpoint will bury a stack trace. Surface their health as a GATE: line instead.
WATCH_PATTERN ?= FINAL_RESULT|GATE:|Error|Traceback|CUDA out of memory

# Entry points. Change these here, not in the recipes.
TRAIN         ?= train.py
EVAL          ?= evaluate.py
PROBE         ?= probe.py

WANDB_FLAG := $(if $(filter 1,$(WANDB)),--wandb --wandb-project $(WANDB_PROJECT) --wandb-name $(EXP),)

.PHONY: help setup brief exp smoke probe eval monitor-cmd status-exp result-exp \
        diag-exp gates-exp logs ps kill-exp guard-EXP

# --- help --------------------------------------------------------------------------
help: ## Show this help
	@echo "$(PROJECT) — experiment driver"
	@echo
	@grep -hE '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) \
	  | awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-16s\033[0m %s\n", $$1, $$2}'
	@echo
	@echo "  Vars: EXP (required for -exp targets), WANDB=1, LOGDIR=$(LOGDIR), PY='$(PY)'"

setup: ## Install dependencies
	@test -f pyproject.toml || { echo "no pyproject.toml yet"; exit 1; }
	uv sync

brief: ## Session bootstrap: Current Best, Ruled Out, Open Questions. Loop step 1
	@test -f hypothesis_log.md || { echo "no hypothesis_log.md — see research-runbook/SETUP.md"; exit 1; }
	@awk '/^## (Current Best|Ruled Out|Open Questions)/,/^---$$/' hypothesis_log.md

# --- guards ------------------------------------------------------------------------
guard-EXP:
	@test -n "$(EXP)" || { \
	  echo "EXP is required. Example: make $(firstword $(MAKECMDGOALS)) EXP=e1-baseline"; exit 1; }

define require
	@test -f $(1) || { \
	  echo ""; \
	  echo "  Not built yet: $(1)"; \
	  echo "  $(2)"; \
	  echo ""; exit 1; }
endef

# --- running -----------------------------------------------------------------------
# Launch these from the agent with the Bash tool and run_in_background:true — never with
# a shell '&', which detaches the job into invisibility (no completion notification).

exp: guard-EXP ## Full run. make exp EXP=<name> [WANDB=1]
	$(call require,$(TRAIN),The training entry point.)
	@$(MAKE) --no-print-directory monitor-cmd EXP=$(EXP)
	$(PY) -u $(TRAIN) --exp $(EXP) $(WANDB_FLAG) $(ARGS) 2>&1 | tee $(LOG)

smoke: guard-EXP ## Cheap sanity run. Always before exp. Checks crash AND diagnostic health
	$(call require,$(TRAIN),The training entry point.)
	@$(MAKE) --no-print-directory monitor-cmd EXP=$(EXP)
	$(PY) -u $(TRAIN) --exp $(EXP) --smoke $(ARGS) 2>&1 | tee $(LOG)

probe: guard-EXP ## Diagnostics only, no training
	$(call require,$(PROBE),Diagnostic probe over the fixed probe set.)
	@$(MAKE) --no-print-directory monitor-cmd EXP=$(EXP)
	$(PY) -u $(PROBE) --exp $(EXP) $(ARGS) 2>&1 | tee $(LOG)

eval: guard-EXP ## Evaluate a checkpoint. make eval EXP=<name> CKPT=<path>
	$(call require,$(EVAL),Evaluation harness.)
	@test -n "$(CKPT)" || { echo "CKPT is required"; exit 1; }
	$(PY) -u $(EVAL) --exp $(EXP) --checkpoint $(CKPT) $(ARGS) 2>&1 | tee $(LOG)

monitor-cmd: guard-EXP ## Print the log path and the Monitor command for a run
	@echo "log:     $(LOG)"
	@echo "monitor: tail -f $(LOG) | grep -E --line-buffered \"$(WATCH_PATTERN)\""
	@echo "         (--line-buffered is required: without it grep block-buffers in the"
	@echo "          pipe and the Monitor stays silent until the buffer flushes)"

# --- inspection (works before any code exists) --------------------------------------
status-exp: guard-EXP ## Tail the log
	@tail -30 $(LOG)

result-exp: guard-EXP ## Last FINAL_RESULT line
	@grep -h "^FINAL_RESULT:" $(LOG) | tail -1 || echo "no FINAL_RESULT in $(LOG)"

diag-exp: guard-EXP ## Last domain diagnostics
	@grep -h "^$(DIAG_PREFIX)_DIAG:" $(LOG) | tail -40 || echo "no $(DIAG_PREFIX)_DIAG in $(LOG)"

gates-exp: guard-EXP ## Every gate with value, threshold, verdict. Non-zero exit if any FAIL
	@grep -h "^GATE:" $(LOG) || { echo "no GATE: lines in $(LOG)"; exit 0; }
	@echo
	@passed=$$(grep -c "verdict=PASS" $(LOG) || true); \
	 failed=$$(grep -c "verdict=FAIL" $(LOG) || true); \
	 echo "  $$passed passed, $$failed failed"; \
	 if [ "$$failed" -gt 0 ]; then \
	   echo "  -> record the failure in hypothesis_log.md before changing anything"; \
	   exit 1; \
	 fi

logs: ## List experiment logs, newest first
	@ls -lt $(LOGDIR)/$(PROJECT)-*.log 2>/dev/null || echo "no logs in $(LOGDIR)"

# --- process management ------------------------------------------------------------
ps: ## Show running training/eval processes
	@pgrep -fl "$(TRAIN)|$(EVAL)|$(PROBE)" || echo "nothing running"

kill-exp: ## Kill all running training/eval processes
	@pkill -f "$(TRAIN)|$(EVAL)|$(PROBE)" && echo "killed" || echo "nothing running"
