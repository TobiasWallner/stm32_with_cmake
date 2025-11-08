# stm32_with_cmake
explains how to create an STM32 project and advance it from a CUBE-IDE project to a CMake project

----

## Create a CUBE-IDE project, that will create some necessarly files, like:
   - Startup files
   - Linker scripts
   - HAL Library


## Create a CMake file

Create a toolchain file that includes the compilers and some micro-processor specific settings.
The following shows a toolchain file that starts out by defining the compiler that is going to be used
and then sets some general project wide public CPU specific flags for the `STM32F446xx`.

It is important that those flags are set publicly, so that all translation units use the same flags and are binary compatible.

```cmake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

# Path to ARM toolchain

set(CMAKE_C_COMPILER "arm-none-eabi-gcc.exe")
set(CMAKE_CXX_COMPILER "arm-none-eabi-g++.exe")
set(CMAKE_ASM_COMPILER "arm-none-eabi-gcc.exe")
set(CMAKE_OBJCOPY "arm-none-eabi-objcopy.exe")
set(CMAKE_SIZE "arm-none-eabi-size.exe")


set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)


###########################################
#		Global Options for all targets
###########################################

set(CPU_FLAGS 
	"-mcpu=cortex-m4"		# use cortex m4 instructions
	"-mthumb" 				# use 16-bit (Thumb) instead of 32-bit (full ARM) instructions | STM32 only supports Thumb
	"-mfpu=fpv4-sp-d16" 	# This specifies what floating-point hardware
	"-mfloat-abi=hard"		# Generate hardware instructions for the floating-point hardware

	# limit the number of errors
	-fmax-errors=3
)

# Set target architecture and cpu specific parameters
#
# Description of the compile options:
#	https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html
#
# PM0214 STM32 Cortex®-M4 MCUs and MPUs programming manual 10.0 
#	https://www.st.com/resource/en/programming_manual/pm0214-stm32-cortexm4-mcus-and-mpus-programming-manual-stmicroelectronics.pdf
#
# RM0390 STM32F446xx advanced Arm®-based 32-bit MCUs 6.0 
# 	https://www.st.com/resource/en/reference_manual/rm0390-stm32f446xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf

add_compile_options(
	# use small c library implementation
	--specs=nano.specs 

	# Place functions and data in separate sections to allow linker garbage collection.
	-ffunction-sections 
	-fdata-sections 
	-fomit-frame-pointer

	-fno-rtti 
	
	#-fno-exceptions
	#-fno-unwind-tables

	${CPU_FLAGS}
)


add_link_options(
	-Wl,--gc-sections 
	-Wl,-T${CMAKE_SOURCE_DIR}/STM32F446RETX_FLASH.ld
	${CPU_FLAGS}
)

link_libraries(
    nosys	# Minimal system stubs
	gcc
    c_nano	# Standard C library
    m		# Math library
)

add_compile_definitions(
    USE_HAL_DRIVER
    STM32F446xx
)
```

## Setup the Main CMakeLists.txt

In the main `CMakeLists.txt` get all source files and header files that include the files that the CUBE-IDE generated for us.
Those include:
 - The startup file
 - The core files (this is where `main.c` is at)
 - The HAL source files

```cmake
###########################################
#			Project Build Setup
###########################################

# HAL, CMSIS, and Core-Initialisation
file(GLOB_RECURSE Core_Src_Files "Core/Src/*.c")
file(GLOB_RECURSE HAL_Src_Files "Drivers/*.c")

set(CORE_SRC_FILES
	Core/Startup/startup_stm32f446retx.s
	${Core_Src_Files}
	${HAL_Src_Files}
)

set(CORE_INC_DIRS
	Core/Inc
	Drivers/CMSIS/Include
	Drivers/CMSIS/Device/ST/STM32F4xx/Include
	Drivers/STM32F4xx_HAL_Driver/Inc
	Drivers/STM32F4xx_HAL_Driver/Inc/Legacy
)
```

Then just create the main executable:

```
add_executable(main ${CORE_SRC_FILES})
```

Optionally link some libraryies or do some fancy CMake stuff and then finally make sure to create an `.elf` and not an `.exe`.

```cmake
set_target_properties(main PROPERTIES OUTPUT_NAME "main.elf")
```

## Furhter you can create a Makefile that helps you build and flash everything:

STM32 offers you a programmer that you can use to flash the binary onto the chip.

```Makefile
#####################################
# 			Variables
#####################################

STM32_Programmer_CLI_PATH = C:/ST/STM32CubeIDE_1.18.0/STM32CubeIDE/plugins/com.st.stm32cube.ide.mcu.externaltools.cubeprogrammer.win32_2.2.100.202412061334/tools/bin
STM32_Programmer_CLI = $(STM32_Programmer_CLI_PATH)/STM32_Programmer_CLI.exe

gcc_size = arm-none-eabi-size
gcc_nm = arm-none-eabi-nm

build_dir = build
config = Release
target = main
test_target = test
linker_script = STM32F446RETX_FLASH.ld

#####################################
# 			Commands
#####################################

.PHONY: help
help: 
	$(info Make Commands:)
	$(info --------------)
	$(info build .......... builds the 'target' with the 'config')
	$(info flash .......... builds the 'target' with the 'config' and then flashes the binary to the micro-controller)
	$(info test ........... convenience command that executest flash with the 'test_target')
	$(info )
	$(info Optional Arguments:)
	$(info ---------------)
	$(info build_dir= ...... sets the build directory. Default: $(build_dir))
	$(info config= ........ sets the build type: 'Release' or 'Debug'. Default: $(config))
	$(info target= ........ sets the target/CMake executable. Default: $(target))
	$(info test_target= ... sets the target/CMake executable. Default: $(test_target))

# CMAKE Setup
# -----------
.PHONY: setup_cmake
setup_cmake:
	$(info )
	$(info ================================ [build_dir: ./$(build_dir)/ | target: $(target) | config: $(config)] ================================)
	$(info )
	cmake -S . -B $(build_dir) -G "Ninja Multi-Config" -DCMAKE_TOOLCHAIN_FILE=toolchain.cmake -DFIBER_COMPILE_TESTS=ON
	
# build commands
# --------------

# build run target
.PHONY: build
build: setup_cmake
	cmake --build $(build_dir) --target $(target) --config $(config)
	$(gcc_size) --format=berkeley "$(build_dir)/$(config)/$(target).elf" > "$(build_dir)/$(config)/$(target)_size.txt"
	python stats.py -l $(linker_script) -s "$(build_dir)/$(config)/$(target)_size.txt"
	$(gcc_nm) --size --print-size --radix=d "$(build_dir)/$(config)/$(target).elf" > "$(build_dir)/$(config)/$(target)_fsize.txt"

# flash run target
.PHONY: flash
flash: build
	${STM32_Programmer_CLI} -c port=SWD -d "$(build_dir)/$(config)/$(target).elf" -v -rst
	
.PHONY: test
test: build
	@make flash target=$(test_target)

.PHONY: openocd
openocd: build_run_debug
	openocd -f openocd.cfg

.PHONY: gdb
gdb:
	arm-none-eabi-gdb "$(build_dir)/$(config)/$(target).elf"


.PHONY: clean
clean:
	cmake --build build --target clean --config Release
	cmake --build build --target clean --config Debug

```

## Add debugging

For debugging you will need to have a config file `openocd.cfg` that looks kinda like:

```
source [find "interface/stlink.cfg"]
source [find "target/stm32f4x.cfg"]

reset_config srst_only
init
reset init
```

Adding debugging support for VSCode you want to create `.vscode/launch.json`:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "STM32 Debug",
            "type": "cortex-debug",
            "request": "attach",
            "servertype": "openocd",
            "configFiles": ["openocd.cfg"],
            "cwd": "${workspaceRoot}",
            "executable": "${workspaceRoot}/build/Debug/main.elf",
            "gdbTarget": "localhost:3333",
            "showDevDebugOutput": "raw",
            "runToEntryPoint": "main",
            "preLaunchTask": "flash-debug"
        }
    ]
}
```

as well as a `.vscode/tasks.json`
```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "flash-debug",
            "type": "shell",
            "command": "make",
            "args": ["flash", "config=Debug"],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": ["$gcc"]
        }
    ]
}
```

This then allows you to run debug builds using `gdb` and skip and jump over lines.
