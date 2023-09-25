# Makefile for the Particle Workbench
# Supports local compile and local flash for Particle projects
#
# Inputs:
# PARTICLE_CLI_PATH: path to the Particle CLI executable
# APPDIR: path to the user application sources
# DEVICE_OS_PATH: path to the source code of the Particle DeviceOS
# PLATFORM: the name of the Particle platform to compile for
# PLATFORM_ID: the id of the Particle platform to compile for

version = 1.15.0
all: flash-all

run_step := $(MAKE) -f $(lastword $(MAKEFILE_LIST))

# Set target build output directory
export TARGET_DIR := $(APPDIR)/target/$(DEVICE_OS_VERSION)/$(PLATFORM)

# Remove env vars which may conflict with our build environment
unexport LDFLAGS
unexport CFLAGS
unexport CXXFLAGS
unexport CPPFLAGS
unexport CONLYFLAGS
unexport ASFLAGS

# Convert Wiring files (ino) files to C++ files (cpp)
INO_FILES := $(wildcard $(APPDIR)/*.ino) $(wildcard $(APPDIR)/src/*.ino)
PREPROCESSED_FILES := $(patsubst %.ino,%.cpp,$(INO_FILES))
inspect-env:
ifeq ($(PARTICLE_LOCAL_COMPILER_DEBUG),1)
	$(call print_stage,OUTPUTTING ENVIRONMENT INFO)
	@env
endif

preprocess: $(PREPROCESSED_FILES)
$(PREPROCESSED_FILES): %.cpp: %.ino
	@"$(PARTICLE_CLI_PATH)" preprocess $< --no-update-check --quiet

# Check if the project has assets, and determine which file to flash
APP_BIN := $(TARGET_DIR)/$(notdir $(APPDIR)).bin
APP_BUNDLE := $(TARGET_DIR)/$(notdir $(APPDIR)).zip
ifeq ($(OS),Windows_NT)
    CHECK_COMMAND = findstr /B assetOtaDir= project.properties > nul
else
    CHECK_COMMAND = grep -q "^assetOtaDir=" project.properties
endif
ifeq ($(wildcard $(APPDIR)/project.properties),)
    HAS_ASSETS := 0
else
    HAS_ASSETS := $(shell $(CHECK_COMMAND) && echo 1 || echo 0)
endif
ifeq ($(HAS_ASSETS),1)
    APP_FLASH := $(APP_BUNDLE)
    TARGET_BUNDLE := $(APP_BUNDLE)
else
    APP_FLASH := $(APP_BIN)
    TARGET_BUNDLE :=
endif

## Top-level targets

# Get rid of compiled files in the user part
clean-user: inspect-env
	$(call print_stage,CLEANING APPLICATION)
	cd "$(DEVICE_OS_PATH)/main" && make clean
	$(call print_success,CLEANED SUCCESSFULLY)

# Get rid of compiled files in the user and system parts
clean-all: inspect-env
	$(call print_stage,CLEANING APPLICATION & DEVICE OS)
	cd "$(DEVICE_OS_PATH)/modules" && make clean
	$(call print_success,CLEANED SUCCESSFULLY)

# Get rid of compiled files for debug build
clean-debug: inspect-env
	$(call print_stage,CLEANING APPLICATION FOR DEBUG)
	cd "$(DEVICE_OS_PATH)/main" && make clean
	$(call print_success,CLEANED SUCCESSFULLY)

# Get rid of everything
sanitize: | inspect-env clean-debug clean-all

# Compile the user part
compile-user: inspect-env
	$(call print_stage,COMPILING APPLICATION)
	$(run_step) compile-user-steps
	$(call print_success,COMPILED SUCCESSFULLY)

compile-user-steps: preprocess make-main create-bundle

# Compile the user and system parts
compile-all: inspect-env
	$(call print_stage,COMPILING APPLICATION & DEVICE OS)
	$(run_step) compile-all-steps
	$(call print_success,COMPILED SUCCESSFULLY)

compile-all-steps: preprocess make-modules create-bundle

# Compile debug build
# NOTE: the steps are now the same as compile user but Compile application for debug command
# needs to stay in Workbench for Gen 2 devices that need to be compiled differently for debug
compile-debug: inspect-env
	$(call print_stage,COMPILING APPLICATION FOR DEBUG)
	$(run_step) compile-user-steps
	$(call print_success,COMPILED SUCCESSFULLY)

# Compile and flash the user part
flash-user: inspect-env
	$(call print_stage,FLASHING APPLICATION)
	$(run_step) flash-user-steps
	$(call print_success,FLASHED SUCCESSFULLY)

flash-user-steps: preprocess make-main create-bundle flash-app

# Compile and flash device-os and bundle or application binary
flash-all: inspect-env
	$(call print_stage,FLASHING APPLICATION & DEVICE OS)
ifeq ($(DEVICE_OS_VERSION),source)
	$(run_step) flash-all-source-steps
else
	$(run_step) flash-all-steps
endif
	$(call print_stage,FLASHING APPLICATION & DEVICE OS)

flash-all-source-steps: preprocess make-modules create-bundle flash-app-and-source-os

flash-all-steps: preprocess make-main create-bundle flash-app-and-os

# Compile and flash debug build
# NOTE: the steps are now the same as flash user but Flash application for debug command
# needs to stay in Workbench for Gen 2 devices that need to be compiled differently for debug
flash-debug: inspect-env
	$(call print_stage,FLASHING APPLICATION)
	$(run_step) flash-user-steps
	$(call print_success,FLASHED SUCCESSFULLY)

## Step targets

# Run Device OS makefile to make the user part
make-main:
	cd "$(DEVICE_OS_PATH)/main" && make all

# Run Device OS makefile to make system and user parts
make-modules:
	cd "$(DEVICE_OS_PATH)/modules" && make all

# Create the bundle if necessary
create-bundle: clear-old-bundle $(TARGET_BUNDLE)

# Get rid of old bundle regardless if a new bundle gets created
clear-old-bundle:
	@rm -f $(APP_BUNDLE)

# Recipe for creating a bundle
$(APP_BUNDLE):
	$(call print_stage,CREATING BUNDLE)
	@cd $(APPDIR) && "$(PARTICLE_CLI_PATH)" bundle $(APP_BIN) --saveTo $(APP_BUNDLE)
	@rm -f $(APP_BIN)

# Flash the application while avoiding to overwrite locally compiled Device OS
ifeq ($(DEVICE_OS_VERSION),source)
FLASH_APP_FLAGS = --application-only
endif
flash-app:
	$(call print_stage,STARTING FLASH)
	@"$(PARTICLE_CLI_PATH)" flash --local $(FLASH_APP_FLAGS) $(PARTICLE_DEVICE_ID) $(APP_FLASH)

# Flash the application and force update Device OS
flash-app-and-os:
	$(call print_stage,STARTING FLASH)
	@"$(PARTICLE_CLI_PATH)" flash --local --target $(DEVICE_OS_VERSION) $(PARTICLE_DEVICE_ID) $(APP_FLASH)

# Flash the application and the locally compiled Device OS files
flash-app-and-source-os:
	$(call print_stage,STARTING FLASH)
	@"$(PARTICLE_CLI_PATH)" flash --local $(PARTICLE_DEVICE_ID) $(APP_FLASH) \
		$(wildcard $(DEVICE_OS_PATH)/build/target/system-part*/platform-$(PLATFORM_ID)-m/system-part*.bin)

help:
	@echo '┌──────────++++'
	@echo '│ Particle Local Compiler'
	@echo '│'
	@echo '│ Current Configuration'
	@echo '│   $$PLATFORM - $(PLATFORM)'
	@echo '│   $$PLATFORM_ID - $(PLATFORM_ID)'
	@echo '│   $$APPDIR - $(APPDIR)'
	@echo '│   $$EXTRA_CFLAGS - $(EXTRA_CFLAGS)'
	@echo '│   $$DEVICE_OS_PATH - $(DEVICE_OS_PATH)'
	@echo '│   $$PARTICLE_DEVICE_ID - $(PARTICLE_DEVICE_ID)'
	@echo '│'
	@echo '│ Available Environment Variables'
	@echo '│   PLATFORM - type of hardware (e.g. photon)'
	@echo '│   PLATFORM_ID - hardware id (e.g. 6)'
	@echo '│   APPDIR - path to your project source files'
	@echo '│   EXTRA_CFLAGS - custom symbols to pass to your application source code'
	@echo '│   DEVICE_OS_PATH - path to device os you are building against'
	@echo '│   PARTICLE_DEVICE_ID - name or id of your device'
	@echo '│   PARTICLE_MAKEFILE - path to main build script'
	@echo '│'
	@echo '│ Available Targets'
	@echo '│   clean-user - clean user application build'
	@echo '│   clean-all - clean user application and device os builds'
	@echo '│   clean-debug - clean debug user application'
	@echo '│   compile-user - compile user application, and creates a bundle if assets are available'
	@echo '│   compile-all - compile user application and device os, and creates a bundle if assets are available'
	@echo '│   compile-debug - compile user application in debug mode, and creates a bundle if assets are available'
	@echo '│   flash-user - flash user application'
	@echo '│   flash-all - flash user application and device os'
	@echo '│   flash-debug - flash user application in debug mode'
	@echo '│'
	@echo '│ Usage'
	@echo '│   make -f $$PARTICLE_MAKEFILE compile-user'
	@echo '│   make -f $$PARTICLE_MAKEFILE compile-user PLATFORM=electron'
	@echo '│   make -f $$PARTICLE_MAKEFILE flash-user EXTRA_CFLAGS="-DFLAG_ONE=abc -DFLAG_TWO=123"'
	@echo '│   DEVICE_OS_PATH=/path/to/device-os make -f $$PARTICLE_MAKEFILE flash-user'
	@echo '│'
	@echo '├─────++++'
	@echo '│ NOTE: THIS TERMINAL WILL RELAUNCH AFTER CHANGING PROJECT SETTINGS'
	@echo '└─────++++'

.PHONY: all inspect-env preprocess clean-user clean-all clean-debug compile-user compile-user-steps compile-all compile-all-steps compile-debug flash-user flash-user-steps flash-all flash-all-source-steps flash-all-steps flash-debug make-main make-modules create-bundle clear-old-bundle flash-app flash-app-and-os flash-app-and-source-os help

define print_stage
	@echo ''
	@echo ':::: $(1)'
	@echo ''
endef

define print_notice
	@echo ''
	@echo '> $(1) <'
	@echo ''
endef

define print_success
	@echo ''
	@echo '*** $(1) ***'
	@echo ''
endef
