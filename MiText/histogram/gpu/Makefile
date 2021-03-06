########################################################
# Rinka Singh/Melt Iron
# Origin: Copyright 1993-2015, NVIDIA Corporation
########################################################

INC_DIR		= ../../../common/include
OBJ_DIR		= ./obj_dir
LIB_DIR		= ../../common/lib/obj_dir
CUDA_PATH       ?= /usr/local/cuda
# dbg = 0
dbg = 1

#for benchmarking and release
# benchmark = 1
#for development Quadro 2000
benchmark = 0

# Debug build flags
ifeq ($(dbg),1)
#      NVDEBUG    := -g -G -O0 --keep		# dump all the intermediate files
#      NVDEBUG    := -g -G -O0 -o0 --ptxas-options=--verbose
      NVDEBUG    := -g -G -O0
      BUILD_TYPE := debug
      DBGFLAGS   := -ggdb -DNDEBUG=1 -o0
      DEBUG      := TRUE
else
      BUILD_TYPE := release
      DBGFLAGS   :=
      DEBUG      :=
endif

LINK_TARGET	= histogram
OBJS		= $(OBJ_DIR)/histogram.o
GPUOBJS		=

LIBOBJS		= $(LIB_DIR)/histo.a
CLEAN_REBUILDABLES = $(OBJS) $(LINK_TARGET)  $(GPUOBJS)
HFILES		= $(INC_DIR)/milib.h $(INC_DIR)/milib_classes.hpp
CUDA_HFILES	= $(INC_DIR)/cuda_milib.h
MAKEFILE	= Makefile

# CUDA Toolkit

##############################
# start deprecated interface #
##############################
ifeq ($(x86_64),1)
    $(info WARNING - x86_64 variable has been deprecated)
    $(info WARNING - please use TARGET_ARCH=x86_64 instead)
    TARGET_ARCH ?= x86_64
endif
ifeq ($(ARMv7),1)
    $(info WARNING - ARMv7 variable has been deprecated)
    $(info WARNING - please use TARGET_ARCH=armv7l instead)
    TARGET_ARCH ?= armv7l
endif
ifeq ($(aarch64),1)
    $(info WARNING - aarch64 variable has been deprecated)
    $(info WARNING - please use TARGET_ARCH=aarch64 instead)
    TARGET_ARCH ?= aarch64
endif
ifeq ($(ppc64le),1)
    $(info WARNING - ppc64le variable has been deprecated)
    $(info WARNING - please use TARGET_ARCH=ppc64le instead)
    TARGET_ARCH ?= ppc64le
endif
ifneq ($(GCC),)
    $(info WARNING - GCC variable has been deprecated)
    $(info WARNING - please use HOST_COMPILER=$(GCC) instead)
    HOST_COMPILER ?= $(GCC)
endif
ifneq ($(abi),)
    $(error ERROR - abi variable has been removed)
endif
############################
# end deprecated interface #
############################

# architecture
HOST_ARCH   := $(shell uname -m)
TARGET_ARCH ?= $(HOST_ARCH)
ifneq (,$(filter $(TARGET_ARCH),x86_64 aarch64 ppc64le))
    TARGET_SIZE := 64
else ifeq ($(TARGET_ARCH),armv7l)
    TARGET_SIZE := 32
else
    $(error ERROR - unsupported value $(TARGET_ARCH) for TARGET_ARCH!)
endif
ifneq ($(TARGET_ARCH),$(HOST_ARCH))
    ifeq (,$(filter $(HOST_ARCH)-$(TARGET_ARCH),aarch64-armv7l x86_64-armv7l x86_64-aarch64 x86_64-ppc64le))
        $(error ERROR - cross compiling from $(HOST_ARCH) to $(TARGET_ARCH) is not supported!)
    endif
endif

# operating system
HOST_OS   := $(shell uname -s 2>/dev/null | tr "[:upper:]" "[:lower:]")
TARGET_OS ?= $(HOST_OS)
ifeq (,$(filter $(TARGET_OS),linux darwin qnx android))
    $(error ERROR - unsupported value $(TARGET_OS) for TARGET_OS!)
endif

# host compiler
ifeq ($(TARGET_OS),darwin)
    ifeq ($(shell expr `xcodebuild -version | grep -i xcode | awk '{print $$2}' | cut -d'.' -f1` \>= 5),1)
        HOST_COMPILER ?= clang++
    endif
else ifneq ($(TARGET_ARCH),$(HOST_ARCH))
    ifeq ($(HOST_ARCH)-$(TARGET_ARCH),x86_64-armv7l)
        ifeq ($(TARGET_OS),linux)
            HOST_COMPILER ?= arm-linux-gnueabihf-g++
        else ifeq ($(TARGET_OS),qnx)
            ifeq ($(QNX_HOST),)
                $(error ERROR - QNX_HOST must be passed to the QNX host toolchain)
            endif
            ifeq ($(QNX_TARGET),)
                $(error ERROR - QNX_TARGET must be passed to the QNX target toolchain)
            endif
            export QNX_HOST
            export QNX_TARGET
            HOST_COMPILER ?= $(QNX_HOST)/usr/bin/arm-unknown-nto-qnx6.6.0eabi-g++
        else ifeq ($(TARGET_OS),android)
            HOST_COMPILER ?= arm-linux-androideabi-g++
        endif
    else ifeq ($(TARGET_ARCH),aarch64)
        ifeq ($(TARGET_OS), linux)
            HOST_COMPILER ?= aarch64-linux-gnu-g++
        else ifeq ($(TARGET_OS), android)
            HOST_COMPILER ?= aarch64-linux-android-g++
        endif
    else ifeq ($(TARGET_ARCH),ppc64le)
        HOST_COMPILER ?= powerpc64le-linux-gnu-g++
    endif
endif
HOST_COMPILER ?= g++
GCC            = gcc
LINT           = /usr/bin/cppcheck
NVCC          := $(CUDA_PATH)/bin/nvcc -ccbin $(HOST_COMPILER)
FLEX           = /usr/bin/flex
BISON          = /usr/bin/bison

# internal flags
NVCCFLAGS   := -m${TARGET_SIZE} $(NVDEBUG)
CFLAGS      :=  -Wall  $(DBGFLAGS) -fstack-protector-all -DBIG_JOINS=1 -fno-strict-aliasing
LFLAGS	    := -L/usr/lib -L/usr/lib/x86_64-linux-gnu -lpthread -lm -ldl -rdc=true
LDFLAGS     :=
CDEFINES    := -DNDEBUG=1

# build flags
ifeq ($(TARGET_OS),darwin)
    LDFLAGS += -rpath $(CUDA_PATH)/lib
    CCFLAGS += -arch $(HOST_ARCH)
else ifeq ($(HOST_ARCH)-$(TARGET_ARCH)-$(TARGET_OS),x86_64-armv7l-linux)
    LDFLAGS += --dynamic-linker=/lib/ld-linux-armhf.so.3
    CCFLAGS += -mfloat-abi=hard
else ifeq ($(TARGET_OS),android)
    LDFLAGS += -pie
    CCFLAGS += -fpie -fpic -fexceptions
endif

ifneq ($(TARGET_ARCH),$(HOST_ARCH))
    ifeq ($(TARGET_ARCH)-$(TARGET_OS),armv7l-linux)
        ifneq ($(TARGET_FS),)
            GCCVERSIONLTEQ46 := $(shell expr `$(HOST_COMPILER) -dumpversion` \<= 4.6)
            ifeq ($(GCCVERSIONLTEQ46),1)
                CCFLAGS += --sysroot=$(TARGET_FS)
            endif
            LDFLAGS += --sysroot=$(TARGET_FS)
            LDFLAGS += -rpath-link=$(TARGET_FS)/lib
            LDFLAGS += -rpath-link=$(TARGET_FS)/usr/lib
            LDFLAGS += -rpath-link=$(TARGET_FS)/usr/lib/arm-linux-gnueabihf
        endif
    endif
endif

ALL_CCFLAGS := --compiler-options -Wall
ALL_CCFLAGS += $(NVCCFLAGS)
ALL_CCFLAGS += $(EXTRA_NVCCFLAGS)
ALL_CCFLAGS += $(addprefix -Xcompiler ,$(CCFLAGS))
ALL_CCFLAGS += $(addprefix -Xcompiler ,$(EXTRA_CCFLAGS))

ALL_LDFLAGS := -lcuda -L$(LIB_DIR)
ALL_LDFLAGS += $(ALL_CCFLAGS)
ALL_LDFLAGS += $(addprefix -Xlinker ,$(LDFLAGS))
ALL_LDFLAGS += $(addprefix -Xlinker ,$(EXTRA_LDFLAGS))

# Common includes and paths for CUDA
INCLUDES  := -I$(INC_DIR)
LIBRARIES := -L$(LIB_OBJS) -L/usr/local/cuda/lib64 -lcudart
#LIBRARIES := -L$(LIB_DIR) -lgpu -L/usr/local/cuda/lib64 -lcudart

################################################################################
SAMPLE_ENABLED := 1

# Gencode arguments
ifeq ($(benchmark),1)
	SMS ?= 20 30 35 37 50 52 60
else
	SMS ?= 20 30 35 37 50 52
endif

ifeq ($(SMS),)
$(info >>> WARNING - no SM architectures have been specified - waiving sample <<<)
SAMPLE_ENABLED := 0
endif

ifeq ($(GENCODE_FLAGS),)
# Generate SASS code for each SM architecture listed in $(SMS)
$(foreach sm,$(SMS),$(eval GENCODE_FLAGS += -gencode arch=compute_$(sm),code=sm_$(sm)))

# Generate PTX code from the highest SM architecture in $(SMS) to guarantee forward-compatibility
HIGHEST_SM := $(lastword $(sort $(SMS)))
ifneq ($(HIGHEST_SM),)
GENCODE_FLAGS += -gencode arch=compute_$(HIGHEST_SM),code=compute_$(HIGHEST_SM)
endif
endif

INCLUDES += -I. -I$(CUDA_PATH)/include/

ifeq ($(SAMPLE_ENABLED),0)
EXEC ?= @echo "[@]"
endif

################################################################################
# Target rules
all: build

build: $(LINK_TARGET)

check.deps:
ifeq ($(SAMPLE_ENABLED),0)
	@echo "Sample will be waived due to the above missing dependencies"
else
	@echo "Sample is ready - all dependencies have been met"
endif
 
$(OBJ_DIR)/histogram.o: histogram.cpp $(HFILES) $(CUDA_HFILES) $(MAKEFILE) $(LIBOBJS)
	$(EXEC) $(NVCC) $(INCLUDES)  $(CCDEFINES) $(ALL_CCFLAGS) $(GENCODE_FLAGS) -o $@ -c $<

$(LINK_TARGET): $(OBJS) $(GPUOBJS) $(LIBOBJS)
	$(EXEC) $(NVCC) $(ALL_LDFLAGS) $(LFLAGS) $(GENCODE_FLAGS) -o $@ $+ $(LIBRARIES)

all: $(LINK_TARGET)

run: build
	$(EXEC) ./$(EXE)

valgrind: $(LINK_TARGET)
	valgrind --tool=memcheck -v --track-origins=yes --leak-check=yes --show-reachable=yes --num-callers=20 --track-fds=yes ./$(OBJS) ./$(GPUOBJS) > valgrind.txt 2>&1

clean:
	rm -f *~ $(CLEAN_REBUILDABLES) *ii core gdb.txt ../include/*~ *gcno valgrind.txt cmd

clean_tilde:
	rm -f *~

clobber: clean

lint:
	$(LINT) --enable=all --inconclusive --std=posix *cpp *cu
