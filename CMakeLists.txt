cmake_minimum_required(VERSION 3.16)
project(PrecisionMouseFilter LANGUAGES CXX RC)

if(NOT WIN32)
    message(WARNING "Precision Mouse Filter targets Windows (Raw Input / Win32 API). "
                     "Configuring anyway for cross-compilation validation only.")
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Add version information
set(PROJECT_VERSION_MAJOR 1)
set(PROJECT_VERSION_MINOR 0)
set(PROJECT_VERSION_PATCH 0)

add_executable(PrecisionMouseFilter WIN32
    src/main.cpp
    src/MouseFilterPipeline.cpp
    src/InputEngine.cpp
    src/ConfigManager.cpp
    src/HotkeyManager.cpp
    src/MetricsMonitor.cpp
    src/MainWindow.cpp
    resources/app.rc
)

target_include_directories(PrecisionMouseFilter PRIVATE include resources)

target_compile_definitions(PrecisionMouseFilter PRIVATE
    UNICODE
    _UNICODE
    WIN32_LEAN_AND_MEAN
    NOMINMAX
    _WIN32_WINNT=0x0A00
    PROJECT_VERSION="${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}"
)

if(MSVC)
    # MSVC compiler options for optimized release build. C++ standard is
    # already set via CMAKE_CXX_STANDARD above; CMake translates that into
    # /std:c++17 automatically, so it is deliberately NOT repeated here (a
    # second, identical /std:c++17 on the command line is harmless in
    # practice, but there's no reason to keep two sources of truth for it).
    target_compile_options(PrecisionMouseFilter PRIVATE
        /O2          # Optimize for speed
        /W4          # Warning level 4
        /EHsc        # Exception handling
    )
else()
    # GCC/Clang compiler options
    target_compile_options(PrecisionMouseFilter PRIVATE -O3 -Wall -Wextra -std=c++17)
    # MinGW's CRT needs to be told explicitly to use the Unicode entry point
    # (wWinMainCRTStartup, which calls our wWinMain); MSVC resolves this
    # automatically and does not need/accept this flag.
    target_link_options(PrecisionMouseFilter PRIVATE -municode -static -static-libgcc -static-libstdc++)
endif()

target_link_libraries(PrecisionMouseFilter PRIVATE
    user32
    gdi32
    comctl32
    comdlg32
    shell32
    ole32
    psapi
    uxtheme
    dwmapi
    advapi32
    shlwapi
)

# Set output directory
set_target_properties(PrecisionMouseFilter PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
)

if(MSVC)
    # The manifest is already embedded manually as resource ID 1 via
    # resources/app.rc ("1 24 \"app.manifest\""). MSVC's linker also embeds
    # its own generated manifest as resource ID 1 by default
    # (/MANIFEST:EMBED,ID=1), which collides with ours and produces
    # CVT1100: duplicate resource. Disable the linker's own manifest
    # generation/embedding since we're supplying it ourselves via the .rc.
    target_link_options(PrecisionMouseFilter PRIVATE /MANIFEST:NO)
endif()
