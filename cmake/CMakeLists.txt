# 1.CMake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_VERSION 1)

# 2.Project
# 设置CMake编译的目标类型为静态库
set(CMAKE_TRY_COMPILE_TARGET_TYPE "STATIC_LIBRARY")
cmake_minimum_required(VERSION 3.20)
project(CH32 C CXX ASM)

# 3.语言标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 11)

# 4.PATH
set(TOOLPATH "D:/program/MounRiver_Studio/toolchain/RISC-V Embedded GCC/bin/riscv-none-embed-")
if (WIN32)
    MESSAGE(STATUS "windows")
    set(CMAKE_C_COMPILER ${TOOLPATH}gcc.exe)
    set(CMAKE_CXX_COMPILER ${TOOLPATH}g++.exe)
    set(CMAKE_ASM_COMPILER ${TOOLPATH}gcc.exe)
    set(CMAKE_AR ${TOOLPATH}ar.exe)
    set(CMAKE_OBJCOPY ${TOOLPATH}objcopy.exe)
    set(CMAKE_OBJDUMP ${TOOLPATH}objdump.exe)
    set(SIZE ${TOOLPATH}size.exe)
elseif (UNIX)
    MESSAGE(STATUS "UNIX-like")
    set(CMAKE_C_COMPILER ${TOOLPATH}gcc)
    set(CMAKE_CXX_COMPILER ${TOOLPATH}g++)
    set(CMAKE_ASM_COMPILER ${TOOLPATH}gcc)
    set(CMAKE_AR ${TOOLPATH}ar)
    set(CMAKE_OBJCOPY ${TOOLPATH}objcopy)
    set(CMAKE_OBJDUMP ${TOOLPATH}objdump)
    set(SIZE ${TOOLPATH}size)
else ()
    MESSAGE(STATUS "Unsupported system!")
endif ()

# 5.编译选项
add_compile_options(
        -march=rv32imac  # 设置目标架构为RISC-V，具有RV32I基础整数指令集，M标准乘法和除法扩展，A标准原子扩展，C标准压缩扩展
        -mabi=ilp32     # 设置应用二进制接口（ABI）为ilp32，即整数、长整数和指针都是32位
        -mcmodel=medany  # 设置代码模型为medany，适用于任何大小的代码和数据
        -msmall-data-limit=8  # 设置小数据区的大小限制为8字节
        -mno-save-restore  # 禁用在函数调用前后保存和恢复寄存器的优化
)
add_compile_options(
        -fmessage-length=0  # 设置诊断消息的最大长度为0，即没有最大长度
        -fsigned-char  # 设置char类型为有符号类型
        -ffunction-sections  # 将每个函数放入自己的section中，以便链接器优化
        -fdata-sections  # 将数据放入自己的section中，以便链接器优化
        -fno-common  # 不允许多个文件共享同一个全局变量
)

# 6.编译等级
if ("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
    message(STATUS "Maximum optimization for speed")
    add_compile_options(-Ofast -flto)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
    message(STATUS "Debug mode")
    add_compile_options(-Og -g)
else ()
    message(STATUS "Minimal optimization, debug info included")
    add_compile_options(-Og -g)
endif ()

# 7.编译信息等级
add_compile_options(-Wall -Wextra -Wpedantic -Wstack-usage=2048)

# 8.编译路径
include_directories(
        ../libraries/sdk/Core
        ../libraries/sdk/Peripheral
        ../libraries/zf_common
        ../libraries/zf_components
        ../libraries/zf_device
        ../libraries/zf_driver
        ../project/code/inc
        ../project/user/inc
)
add_definitions(-DDEBUG=1)
file(GLOB_RECURSE SOURCES
        "${CMAKE_SOURCE_DIR}/../libraries/sdk/Startup/*.*"
        "${CMAKE_SOURCE_DIR}/../libraries/*.c"
        "${CMAKE_SOURCE_DIR}/../libraries/*.cpp"
        "${CMAKE_SOURCE_DIR}/../libraries/*.cc"
        "${CMAKE_SOURCE_DIR}/../project/*.c"
        "${CMAKE_SOURCE_DIR}/../project/*.cpp"
        "${CMAKE_SOURCE_DIR}/../project/*.cc"
)
# list(REMOVE_ITEM SOURCES ${CMAKE_SOURCE_DIR}/FreeRTOS/portable/Common/mpu_wrappers.c)
set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/../libraries/sdk/Ld/Link.ld)

# 9.链接选项
add_link_options(
        -march=rv32imac  # 设置目标架构为RISC-V，具有RV32I基础整数指令集，M标准乘法和除法扩展，A标准原子扩展，C标准压缩扩展
        -mabi=ilp32  # 设置应用二进制接口（ABI）为ilp32，即整数、长整数和指针都是32位
        -nostartfiles  # 不使用标准系统启动文件
        -Xlinker --gc-sections  # 通过链接器选项--gc-sections，删除未使用的代码和数据块
        -Wl,--print-memory-usage  # 通过链接器选项--print-memory-usage，打印内存使用情况
        -Wl,-Map,${PROJECT_NAME}.map  # 通过链接器选项-Map，生成map文件，显示符号的分布情况
        --specs=nano.specs  # 使用nano.specs规格文件，这会链接更小的C库
        --specs=nosys.specs  # 使用nosys.specs规格文件，这会链接一个不包含系统调用的C库
)
add_link_options(-T ${LINKER_SCRIPT})

# 10.生成固件
add_executable(${PROJECT_NAME}.elf ${SOURCES} ${LINKER_SCRIPT})
# 静态
# target_link_libraries(${PROJECT_NAME}.elf printfloat)

# 11.输出文件
set(HEX_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin)
set(LST_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.lst)
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE} # .hex
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE} # .bin
        COMMAND ${CMAKE_OBJDUMP} --all-headers --demangle --disassemble $<TARGET_FILE:${PROJECT_NAME}.elf> > ${LST_FILE} # .lst
        COMMENT "Building ${HEX_FILE}\nBuilding ${BIN_FILE}"
)