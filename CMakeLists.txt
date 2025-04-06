cmake_minimum_required(VERSION 3.22)

# 交叉编译工具链配置 (不分离版本)
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_VERSION 1)

# 设置工具链路径 (可通过 -DGCC_PATH= 参数指定)
if (GCC_PATH)
    # 统一处理路径分隔符和可执行文件后缀
    file(TO_CMAKE_PATH "${GCC_PATH}" GCC_PATH)
    if (WIN32)
        set(TOOL_SUFFIX ".exe")
        # 确保路径以反斜杠结尾
        if(NOT GCC_PATH MATCHES ".*\\$")
            set(GCC_PATH "${GCC_PATH}\\")
        endif()
    else()
        set(TOOL_SUFFIX "")
        # 确保路径以斜杠结尾
        if(NOT GCC_PATH MATCHES ".*/$")
            set(GCC_PATH "${GCC_PATH}/")
        endif()
    endif()

    set(CMAKE_C_COMPILER "${GCC_PATH}arm-none-eabi-gcc${TOOL_SUFFIX}")
    set(CMAKE_CXX_COMPILER "${GCC_PATH}arm-none-eabi-g++${TOOL_SUFFIX}")
    set(CMAKE_ASM_COMPILER "${GCC_PATH}arm-none-eabi-gcc${TOOL_SUFFIX}")
    set(CMAKE_AR "${GCC_PATH}arm-none-eabi-ar${TOOL_SUFFIX}")
    set(CMAKE_OBJCOPY "${GCC_PATH}arm-none-eabi-objcopy${TOOL_SUFFIX}")
    set(CMAKE_OBJDUMP "${GCC_PATH}arm-none-eabi-objdump${TOOL_SUFFIX}")
    set(SIZE "${GCC_PATH}arm-none-eabi-size${TOOL_SUFFIX}")
else()
    # 使用系统路径中的工具链
    set(CMAKE_C_COMPILER arm-none-eabi-gcc)
    set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
    set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
    set(CMAKE_AR arm-none-eabi-ar)
    set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
    set(CMAKE_OBJDUMP arm-none-eabi-objdump)
    set(SIZE arm-none-eabi-size)
endif()

# 验证编译器是否存在
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# 项目配置
project(LE5010_cmake LANGUAGES C ASM)

# 默认构建类型
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build type" FORCE)
endif()

# 标准设置
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)


set(CMAKE_BUILD_TYPE Debug)

set(COMPILER gnu)
add_compile_options(
        -mabi=aapcs -mthumb -mcpu=cortex-m0 -mno-unaligned-access
        -fshort-enums -fshort-wchar
        -g -ffunction-sections -fdata-sections
        -fstrict-volatile-bitfields -fno-common
        -Wall -Os
)

# 公共编译选项
add_compile_options(
        -mcpu=cortex-m0
        -mthumb
        -mthumb-interwork
        -ffunction-sections
        -fdata-sections
        -fno-common
        -fmessage-length=0
)

# 构建类型特定选项
string(TOLOWER "${CMAKE_BUILD_TYPE}" BUILD_TYPE_LOWER)
if(BUILD_TYPE_LOWER STREQUAL "release")
    add_compile_options(-Ofast)
elseif(BUILD_TYPE_LOWER STREQUAL "relwithdebinfo")
    add_compile_options(-Ofast -g)
elseif(BUILD_TYPE_LOWER STREQUAL "minsizerel")
    add_compile_options(-Os)
else()
    # Debug 和其他情况
    add_compile_options(-Og -c)
endif()

# 源文件收集
set(SOURCE_DIRS
        soc/arm_cm
        soc/arm_cm/le501x
        soc/arm_cm/le501x/compiler/gnu
        soc/arm_cm/le501x/integration
        hal_driver/src
        module
        module/common
        module/crc
        module/debug
        module/debug/RTT
        module/static_buffer
        module/tinyfs
        module/tinycrypt
#        module/ble
        module/settings
        examples/ble/ble_uart_server
)

set(SOURCES)
foreach(DIR ${SOURCE_DIRS})
    file(GLOB_RECURSE DIR_SOURCES
            "${CMAKE_CURRENT_SOURCE_DIR}/${DIR}/*.c"
            "${CMAKE_CURRENT_SOURCE_DIR}/${DIR}/*.s"
            "${CMAKE_CURRENT_SOURCE_DIR}/${DIR}/*.S"
    )
    # 移除不需要的文件
    # SDK里的例子文件太细了，我用了排除法
    list(FILTER SOURCES EXCLUDE REGEX ".*[/\\\\]armcc|iar|ble|tests|test|source[/\\\\].*")
    list(FILTER SOURCES EXCLUDE REGEX "port.c")
    list(FILTER SOURCES EXCLUDE REGEX "\\.inc")
    list(FILTER SOURCES EXCLUDE REGEX ".*[/\\\\]freertos[^/\\\\]*\\.(c|s|S)$")
    list(APPEND SOURCES ${DIR_SOURCES})
endforeach()

list(APPEND SOURCES "${PROJECT_SOURCE_DIR}/module/ble/ble_common_api.c")
# 添加启动文件
list(APPEND SOURCES "${PROJECT_SOURCE_DIR}/soc/arm_cm/le501x/compiler/${COMPILER}/startup.S")

# 添加不开源的库
list(APPEND SOURCES "${PROJECT_SOURCE_DIR}/soc/arm_cm/le501x/bin/libfw.o")
# 创建可执行目标
add_executable(${PROJECT_NAME}.elf ${SOURCES})

# 包含目录 (使用现代target_include_directories)
target_include_directories(${PROJECT_NAME}.elf PUBLIC
        ${PROJECT_SOURCE_DIR}/hal_driver/inc
        ${PROJECT_SOURCE_DIR}/hal_driver/api
        ${PROJECT_SOURCE_DIR}/module/common
        ${PROJECT_SOURCE_DIR}/module/crc
        ${PROJECT_SOURCE_DIR}/module/debug
        ${PROJECT_SOURCE_DIR}/module/debug/RTT
        ${PROJECT_SOURCE_DIR}/module/static_buffer
        ${PROJECT_SOURCE_DIR}/module/tinyfs
        ${PROJECT_SOURCE_DIR}/module/tinycrypt/lib/include
        ${PROJECT_SOURCE_DIR}/module/micro-ecc
        ${PROJECT_SOURCE_DIR}/module/ble
        ${PROJECT_SOURCE_DIR}/module/ble/prf
        ${PROJECT_SOURCE_DIR}/module/settings
        ${PROJECT_SOURCE_DIR}/soc
        ${PROJECT_SOURCE_DIR}/soc/arm_cm
        ${PROJECT_SOURCE_DIR}/soc/arm_cm/cmsis
        ${PROJECT_SOURCE_DIR}/soc/arm_cm/le501x
        ${PROJECT_SOURCE_DIR}/soc/arm_cm/le501x/integration
        ${PROJECT_SOURCE_DIR}/examples/ble/ble_uart_server
)

# 宏定义
target_compile_definitions(${PROJECT_NAME}.elf PUBLIC
        LE501X
)

# 链接配置
set(LINKER_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/soc/arm_cm/le501x/compiler/${COMPILER}/ble_common.ld")

target_link_options(${PROJECT_NAME}.elf PUBLIC
        -mcpu=cortex-m0 -g -specs=nosys.specs -specs=nano.specs
        -Wl,--print-memory-usage
        -Wl,-Map=${PROJECT_NAME}.map
        -Wl,--fatal-warnings
        -Wl,--cref
        -Wl,--no-wchar-size-warning
        -Wl,--no-enum-size-warning
        -Wl,--gc-sections
        -T "${LINKER_SCRIPT}"
)

# 生成 HEX 和 BIN 文件、
#arm-none-eabi-objdump -d -z -x build/le501x/ble_uart_server.elf > build/le501x/ble_uart_server.asm
#arm-none-eabi-objcopy -O binary build/le501x/ble_uart_server.elf build/le501x/ble_uart_server.ota.bin
#arm-none-eabi-objcopy -O ihex build/le501x/ble_uart_server.elf build/le501x/ble_uart_server.hex
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${PROJECT_NAME}.hex
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${PROJECT_NAME}.bin
        COMMENT "Generating ${PROJECT_NAME}.hex and ${PROJECT_NAME}.bin"
        VERBATIM
)
#生成info_sbl.hex
#402866176 16进制 0x18034000
#python tools/le501x/info_sbl_merge.py soc/arm_cm/le501x/bin/bram.bin 402866176 build/le501x/info_sbl.hex
#python hexmerge.py info_sbl.hex ble_uart_server.hex soc/arm_cm/le501x/bin/fw.hex > build/le501x/ble_uart_server_production.hex
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND python3 ${CMAKE_CURRENT_SOURCE_DIR}/tools/le501x/info_sbl_merge.py ${CMAKE_CURRENT_SOURCE_DIR}/soc/arm_cm/le501x/bin/bram.bin 402866176 info_sbl.hex
        COMMAND python3 ${CMAKE_CURRENT_SOURCE_DIR}/tools/hexmerge.py info_sbl.hex ${PROJECT_NAME}.hex ${CMAKE_CURRENT_SOURCE_DIR}/soc/arm_cm/le501x/bin/fw.hex > ${PROJECT_NAME}_production.hex
        COMMENT "Generating info_sbl.hex and ${PROJECT_NAME}_production.hex"
        VERBATIM
)

# 烧录目标 (可选)
#find_program(PYOCD pyocd)
#if(PYOCD)
#    add_custom_target(flash
#            COMMAND ${PYOCD} flash $<TARGET_FILE:${PROJECT_NAME}.elf> --target LE501X
#            DEPENDS ${PROJECT_NAME}.elf
#            COMMENT "Flashing ${PROJECT_NAME}.elf to target device"
#            VERBATIM
#    )
#else()
#    message(STATUS "pyocd not found, flash target will not be available")
#endif()

# 添加 size 目标用于查看内存使用
add_custom_target(size
        COMMAND ${SIZE} $<TARGET_FILE:${PROJECT_NAME}.elf>
        DEPENDS ${PROJECT_NAME}.elf
        COMMENT "Size of ${PROJECT_NAME}.elf:"
        VERBATIM
)