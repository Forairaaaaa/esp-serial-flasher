cmake_minimum_required(VERSION 3.5)

set(srcs
    src/esp_targets.c
    src/md5_hash.c
    src/esp_loader.c
    src/protocol_common.c
)

if (DEFINED ESP_PLATFORM)
    if (${CONFIG_SERIAL_FLASHER_INTERFACE_UART})
        list(APPEND srcs
            src/protocol_uart.c
            src/slip.c
            port/esp32_port.c
        )
    elseif (${CONFIG_SERIAL_FLASHER_INTERFACE_SPI})
        list(APPEND srcs
            src/protocol_spi.c
            port/esp32_spi_port.c
        )
    endif()

    # Register component to esp-idf build system
    idf_component_register(SRCS ${srcs}
                           INCLUDE_DIRS include port
                           PRIV_INCLUDE_DIRS private_include
                           PRIV_REQUIRES driver esp_timer)

    set(target ${COMPONENT_LIB})
    component_compile_options(-Wstrict-prototypes)

else()
    if (NOT DEFINED SERIAL_FLASHER_INTERFACE_UART AND NOT DEFINED SERIAL_FLASHER_INTERFACE_SPI)
        set(SERIAL_FLASHER_INTERFACE_UART true)
    endif()

    if (DEFINED SERIAL_FLASHER_INTERFACE_UART)
        list(APPEND srcs
            src/protocol_uart.c
            src/slip.c
        )
    elseif (DEFINED SERIAL_FLASHER_INTERFACE_SPI)
        list(APPEND srcs src/protocol_spi.c)
    endif()

    # Create traditional CMake target
    add_library(flasher ${srcs})

    target_include_directories(flasher PUBLIC include port PRIVATE private_include)

    if (NOT DEFINED PORT)
        message(WARNING "No port selected, default to user-defined")
        set(PORT "USER_DEFINED")
    elseif(PORT STREQUAL "USER_DEFINED")
        # The user has to manually link their port with the flasher target
    elseif(PORT STREQUAL "STM32")
        stm32_get_chip_info(${STM32_CHIP} FAMILY DEVICE_FAMILY DEVICE DEVICE_CODE)
        if(DEFINED CORE_USED)
            string(APPEND DEVICE_FAMILY ::${CORE_USED})
            string(APPEND DEVICE_CODE ::${CORE_USED})
        endif()
        target_link_libraries(flasher PRIVATE
            HAL::STM32::${DEVICE_FAMILY}::GPIO
            HAL::STM32::${DEVICE_FAMILY}::UART
            CMSIS::STM32::${DEVICE_CODE}
        )
        target_sources(flasher PRIVATE port/stm32_port.c)
    elseif(PORT STREQUAL "RASPBERRY_PI")
        find_library(pigpio_LIB pigpio)
        target_link_libraries(flasher PUBLIC ${pigpio_LIB})
        target_sources(flasher PRIVATE port/raspberry_port.c)
    else()
        message(FATAL_ERROR "Selected port is not supported")
    endif()

    set(target flasher)

endif()

if (DEFINED SERIAL_FLASHER_INTERFACE_UART OR CONFIG_SERIAL_FLASHER_INTERFACE_UART STREQUAL "y")
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_INTERFACE_UART
    )
    if (DEFINED MD5_ENABLED OR CONFIG_SERIAL_FLASHER_MD5_ENABLED)
        target_compile_definitions(${target} PUBLIC MD5_ENABLED=1)
    endif()
elseif (DEFINED SERIAL_FLASHER_INTERFACE_SPI OR CONFIG_SERIAL_FLASHER_INTERFACE_SPI STREQUAL "y")
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_INTERFACE_SPI
    )
endif()

if(DEFINED SERIAL_FLASHER_DEBUG_TRACE OR CONFIG_SERIAL_FLASHER_DEBUG_TRACE)
    target_compile_definitions(${target} PUBLIC SERIAL_FLASHER_DEBUG_TRACE)
endif()

if (DEFINED CONFIG_SERIAL_FLASHER_RESET_HOLD_TIME_MS)
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_RESET_HOLD_TIME_MS=${CONFIG_SERIAL_FLASHER_RESET_HOLD_TIME_MS}
    )
elseif (DEFINED SERIAL_FLASHER_RESET_HOLD_TIME_MS)
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_RESET_HOLD_TIME_MS=${SERIAL_FLASHER_RESET_HOLD_TIME_MS}
    )
else()
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_RESET_HOLD_TIME_MS=100
    )
endif()

if (DEFINED CONFIG_SERIAL_FLASHER_BOOT_HOLD_TIME_MS)
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_BOOT_HOLD_TIME_MS=${CONFIG_SERIAL_FLASHER_BOOT_HOLD_TIME_MS}
    )
elseif (DEFINED SERIAL_FLASHER_BOOT_HOLD_TIME_MS)
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_BOOT_HOLD_TIME_MS=${SERIAL_FLASHER_BOOT_HOLD_TIME_MS}
    )
else()
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_BOOT_HOLD_TIME_MS=50
    )
endif()

if (DEFINED CONFIG_SERIAL_FLASHER_WRITE_BLOCK_RETRIES)
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_WRITE_BLOCK_RETRIES=${CONFIG_SERIAL_FLASHER_WRITE_BLOCK_RETRIES}
    )
elseif (DEFINED SERIAL_FLASHER_WRITE_BLOCK_RETRIES)
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_WRITE_BLOCK_RETRIES=${SERIAL_FLASHER_WRITE_BLOCK_RETRIES}
    )
else()
    target_compile_definitions(${target}
    PUBLIC
        SERIAL_FLASHER_WRITE_BLOCK_RETRIES=3
    )
endif()
