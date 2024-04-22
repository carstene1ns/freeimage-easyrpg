cmake_minimum_required(VERSION 3.16)
project(FreeImage VERSION 3.18.0 LANGUAGES C CXX)

# Options for configuring the build
option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(BUILD_TESTS "Build the test suite" OFF)

find_package(ZLIB REQUIRED)
find_package(PNG REQUIRED)

# Define the FreeImage library
add_library(${PROJECT_NAME} STATIC
    # FreeImage itself
    Source/FreeImage/BitmapAccess.cpp
    Source/FreeImage/ColorLookup.cpp
    Source/FreeImage/ConversionRGBA16.cpp
    Source/FreeImage/ConversionRGBAF.cpp
    Source/FreeImage/FreeImage.cpp
    Source/FreeImage/FreeImageC.c
    Source/FreeImage/FreeImageIO.cpp
    Source/FreeImage/GetType.cpp
    Source/FreeImage/LFPQuantizer.cpp
    Source/FreeImage/MemoryIO.cpp
    Source/FreeImage/PixelAccess.cpp
    Source/FreeImage/Plugin.cpp
    Source/FreeImage/PluginBMP.cpp
    Source/FreeImage/PluginPNG.cpp
    Source/FreeImage/Conversion.cpp
    Source/FreeImage/Conversion16_555.cpp
    Source/FreeImage/Conversion16_565.cpp
    Source/FreeImage/Conversion24.cpp
    Source/FreeImage/Conversion32.cpp
    Source/FreeImage/Conversion4.cpp
    Source/FreeImage/Conversion8.cpp
    Source/FreeImage/ConversionFloat.cpp
    Source/FreeImage/ConversionRGB16.cpp
    Source/FreeImage/ConversionRGBF.cpp
    Source/FreeImage/ConversionType.cpp
    Source/FreeImage/ConversionUINT16.cpp
    Source/FreeImage/Halftoning.cpp
    Source/FreeImage/tmoColorConvert.cpp
    Source/FreeImage/tmoDrago03.cpp
    Source/FreeImage/tmoFattal02.cpp
    Source/FreeImage/tmoReinhard05.cpp
    Source/FreeImage/ToneMapping.cpp
    Source/FreeImage/NNQuantizer.cpp
    Source/FreeImage/WuQuantizer.cpp
    Source/FreeImage/CacheFile.cpp
    Source/FreeImage/MultiPage.cpp
    Source/FreeImage/ZLibInterface.cpp

    # FreeImageToolkit
    Source/FreeImageToolkit/Background.cpp
    Source/FreeImageToolkit/BSplineRotate.cpp
    Source/FreeImageToolkit/Channels.cpp
    Source/FreeImageToolkit/ClassicRotate.cpp
    Source/FreeImageToolkit/Colors.cpp
    Source/FreeImageToolkit/CopyPaste.cpp
    Source/FreeImageToolkit/Display.cpp
    Source/FreeImageToolkit/Flip.cpp
    Source/FreeImageToolkit/MultigridPoissonSolver.cpp
    Source/FreeImageToolkit/Rescale.cpp
    Source/FreeImageToolkit/Resize.cpp

    # Metadata
    Source/Metadata/FIRational.h
    Source/Metadata/FreeImageTag.h
    Source/Metadata/FIRational.cpp
    Source/Metadata/FreeImageTag.cpp
    Source/Metadata/IPTC.cpp
    Source/Metadata/TagConversion.cpp
    Source/Metadata/TagLib.cpp

    # Utilities
    Source/MapIntrospector.h
    Source/CacheFile.h
    Source/FreeImage.h
    Source/FreeImageIO.h
    Source/Plugin.h
    Source/Quantizers.h
    Source/ToneMapping.h
    Source/Utilities.h
)
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})
target_link_libraries(${PROJECT_NAME} PRIVATE PNG::PNG ZLIB::ZLIB)

# FreeImage's include directories
target_include_directories(${PROJECT_NAME} PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/Source>)
target_include_directories(${PROJECT_NAME} PRIVATE Source/Metadata Source/FreeImageToolkit)

# Preprocessor definitions
if (NOT BUILD_SHARED_LIBS)
    target_compile_definitions(${PROJECT_NAME} PUBLIC FREEIMAGE_LIB)
endif ()

# Set C++ standard
target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_11)
set_target_properties(${PROJECT_NAME} PROPERTIES CXX_EXTENSIONS OFF)

# Emscripten support
if(EMSCRIPTEN)
    add_executable(FreeImage_wasm)
    target_link_libraries(FreeImage_wasm PRIVATE ${PROJECT_NAME}::${PROJECT_NAME})
endif()

# Apple-specific settings
if (APPLE)
    set_target_properties(${PROJECT_NAME} PROPERTIES XCODE_ATTRIBUTE_ONLY_ACTIVE_ARCH "NO")
    if(BUILD_PLATFORM_APPLE_IOS)
        set_target_properties(${PROJECT_NAME} PROPERTIES XCODE_ATTRIBUTE_GCC_THUMB_SUPPORT "NO")
    endif()
endif()

# Enable Multi-core compilation in MSVC
if (MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE /MP)
endif()

# Append -d to debug build so we can have both debug and release builds in the same directory
set_target_properties(${PROJECT_NAME} PROPERTIES DEBUG_POSTFIX -d)

# Test suite
if (BUILD_TESTS)
    include(CTest)
    enable_testing()
    add_executable(MainTestSuite 
        TestAPI/MainTestSuite.cpp
        TestAPI/testChannels.cpp
        TestAPI/testHeaderOnly.cpp
        TestAPI/testImageType.cpp
        TestAPI/testMemIO.cpp
        TestAPI/testPlugins.cpp
        TestAPI/testTools.cpp
        TestAPI/testWrappedBuffer.cpp
    )

    target_link_libraries(MainTestSuite FreeImage::FreeImage)
    add_test(NAME MainTestSuite
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/TestAPI
        COMMAND $<TARGET_FILE:MainTestSuite>)
endif ()

include(GNUInstallDirs)

# Install the library
install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Targets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
install(FILES Source/FreeImage.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# Install the CMake Targets
install(EXPORT ${PROJECT_NAME}Targets
    FILE ${PROJECT_NAME}Targets.cmake
    NAMESPACE ${PROJECT_NAME}::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# Install the CMake config
include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${PROJECT_SOURCE_DIR}/${PROJECT_NAME}Config.cmake.in
    ${PROJECT_NAME}Config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)
write_basic_package_version_file(
    ${PROJECT_NAME}ConfigVersion.cmake
    VERSION ${PACKAGE_VERSION}
    COMPATIBILITY SameMajorVersion
)
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# pkg-config
configure_file(${PROJECT_NAME}.pc.in ${PROJECT_NAME}.pc @ONLY)
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)
