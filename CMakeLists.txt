#================================================================================
cmake_minimum_required(VERSION 3.20)

set(CMAKE_TOOLCHAIN_FILE cmake/arm-none-eabi-gcc.cmake CACHE FILEPATH "" FORCE)


#================================================================================
project(abox LANGUAGES C CXX ASM)

include(FetchContent)  # Используем для работы с зависимостями в форме исходного кода


#--- Внешние зависимости проекта ---
#--- libopencm3 -----------------------------------------------------------------
FetchContent_Declare(libopencm3_repo
  GIT_REPOSITORY https://github.com/libopencm3/libopencm3.git
  GIT_TAG        44e142d4f97863e669737707a1a22bf40ed49bbc  # У libopencm3 есть проблемы с версиями, используем хэш коммита.
)

FetchContent_GetProperties(libopencm3_repo)
if(NOT libopencm3_repo_POPULATED)
     FetchContent_Populate(libopencm3_repo)
endif()


# Libopencm3 пока не дружит с cmake, собираем при помощи make и формируем название цели для сборки перректально, увы.
if(${TARGET_SERIE} STREQUAL "stm32f4")
  set(LIBOPENCM3_TARGET "stm32/f4")  # Архитектура целевого МК
endif()


add_custom_target(libopencm3_make ALL
     COMMAND make TARGETS='${LIBOPENCM3_TARGET}' WORKING_DIRECTORY ${libopencm3_repo_SOURCE_DIR}
)

add_library(libopencm3 INTERFACE)
add_dependencies(libopencm3 libopencm3_make)
target_include_directories(libopencm3 INTERFACE  ${libopencm3_repo_SOURCE_DIR}/include)
target_link_libraries(libopencm3 INTERFACE ${libopencm3_repo_SOURCE_DIR}/lib/libopencm3_${TARGET_SERIE}.a)

string(TOUPPER ${TARGET_SERIE} TARGET_LIBOPENCM3_DEFINE)
#--------------------------------------------------------------------------------


#================================================================================
#--------------------------------------------------------------------------------
add_executable(${PROJECT_NAME}
     src/main.cpp
)

set_property(TARGET ${PROJECT_NAME} PROPERTY CXX_STANDARD 17)
set_property(TARGET ${PROJECT_NAME} PROPERTY C_STANDARD 11)

target_include_directories(${PROJECT_NAME} PRIVATE
     src     
)

target_compile_definitions(${PROJECT_NAME} PRIVATE
     -D${TARGET_LIBOPENCM3_DEFINE}
)

target_link_libraries(${PROJECT_NAME}
     libopencm3
)

target_link_options(${PROJECT_NAME} PRIVATE
     -T${CMAKE_SOURCE_DIR}/conf/stm32f407vgt6.ld
)



#================================================================================
set(OOCD_FLASH_SCRIPT
     "MESSAGE(STATUS \"Flashing firmware\")
     execute_process (
          COMMAND openocd
#               -f ${CMAKE_SOURCE_DIR}/conf/openocd_discovery.cfg
               -f ${CMAKE_SOURCE_DIR}/conf/openocd_stlink.cfg
               -c \"init\" 
               -c \"reset halt\"
               -c \"program ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_master.elf\" 
               -c \"reset\"
               -c \"shutdown\"
     )
     MESSAGE(STATUS \"Flashing complete\")
     "
)

install(CODE
     ${OOCD_FLASH_SCRIPT}
)
