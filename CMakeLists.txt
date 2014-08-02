cmake_minimum_required(VERSION 3.0)

project(
    liblcms2 
    VERSION 2.6
    LANGUAGES C
)

include(FeatureSummary)

set(LCMS_VERSION_MAJOR 2)
set(LCMS_VERSION_MINOR 6)
set(LCMS_VERSION_STR "${LCMS_VERSION_MAJOR}.${LCMS_VERSION_MINOR}")

option (BUILD_SHARED_LIBS "Build Shared Libraries" ON) 
option (BUILD_TOOLS "Build Tools (most depend on JPEG & TIFF)" ON) 
option (BUILD_TESTS "Build Tests" ON) 

if (MSVC)
    if (NOT DEFINED ARCH)
        set (ARCH ${MSVC_C_ARCHITECTURE_ID})
    endif()

    # use boost convention - you cant' get the VC compiler without VS
    if (NOT DEFINED MSVCVER)
        math(EXPR MSVCVER "(${MSVC_VERSION} / 10) - 60")
    endif()
    
    # add the runtime version and target architecture into the DLL name per MS
    # guidelines
    if ("${ARCH}" STREQUAL "x64")
        set(MSVC_SHARED_SUFFIX_BASE "-vc${MSVCVER}-mt-${ARCH}")
    else()
        set(MSVC_SHARED_SUFFIX_BASE "-vc${MSVCVER}-mt")
    endif()
    set(CMAKE_SHARED_LIBRARY_SUFFIX "${MSVC_SHARED_SUFFIX_BASE}.dll")
endif()


set(LIBTARGET "lcms2")

include_directories("${CMAKE_CURRENT_SOURCE_DIR}/include")

file(GLOB SRCS src/*.c)
file(GLOB HDRS include/*.h)

if (WIN32)
    set(runtime_name ${LIBTARGET}${MSVC_SHARED_SUFFIX_BASE})
    
    set(VER_PRODUCTNAME_STR "LittleCMS color engine")
    set(VER_ORIGINALFILENAME_STR "${runtime_name}.dll")
    set(VER_INTERNALNAME_STR ${LIBTARGET})
    set(VER_FILEDESCRIPTION_STR "LittleCMS Dynamic Link Library")
    set(VER_LEGALCOPYRIGHT_STR "Copyright (c) Marti Maria 2011.")
    set(VER_STR ${LCMS_VERSION_STR})
    set(VER_NUM "${LCMS_VERSION_MAJOR},${LCMS_VERSION_MINOR},0,0")
    configure_file(
          ${CMAKE_CURRENT_SOURCE_DIR}/src/lcms2.rc.in
          ${CMAKE_CURRENT_BINARY_DIR}/lcms2.rc
          @ONLY
      )
    set(VCSHAREDFILES src/lcms2.def ${CMAKE_CURRENT_BINARY_DIR}/lcms2.rc)
else(WIN32)
    set(runtime_name ${LIBTARGET})
endif()

if (BUILD_SHARED_LIBS)
    add_library(${LIBTARGET} SHARED ${SRCS} ${HDRS} ${VCSHAREDFILES})    
endif()
 
set_target_properties(
    ${LIBTARGET}
        PROPERTIES 
          SOVERSION ${LCMS_VERSION_MAJOR}.${LCMS_VERSION_MINOR}
          VERSION ${LCMS_VERSION_MAJOR}
          PUBLIC_HEADER "${HDRS}"
  )


if (BUILD_TOOLS)
    find_package(JPEG)
    find_package(TIFF)

    set (tools_common 
        utils/common/utils.h
        utils/common/vprf.c
        utils/common/xgetopt.c
    )
    
    # linkicc
    add_executable(linkicc
        ${tools_common}
        utils/linkicc/linkicc.c
    )
    target_link_libraries(linkicc PRIVATE ${LIBTARGET})
    target_include_directories(linkicc PRIVATE utils/common)
   
    add_executable(psicc
        ${tools_common}
        utils/psicc/psicc.c
    )
    target_link_libraries(psicc PRIVATE ${LIBTARGET})
    target_include_directories(psicc PRIVATE utils/common)

    add_executable(transicc
        ${tools_common}
        utils/transicc/transicc.c
    )
    target_link_libraries(transicc PRIVATE ${LIBTARGET})
    target_include_directories(transicc PRIVATE utils/common)
    

    set (tool_targets psicc linkicc transicc)

    if (JPEG_FOUND)
        add_executable(jpgicc 
            ${tools_common}
            utils/jpgicc/iccjpeg.c
            utils/jpgicc/iccjpeg.h
            utils/jpgicc/jpgicc.c
        )
        target_link_libraries(jpgicc PRIVATE ${LIBTARGET} ${JPEG_LIBRARIES})
        target_include_directories(jpgicc PRIVATE ${JPEG_INCLUDE_DIR} utils/common)

        set (tool_targets ${tool_targets} jpgicc)

    endif(JPEG_FOUND)

    if (TIFF_FOUND)
        add_executable(tificc
            ${tools_common}
            utils/tificc/tificc.c
            )
        target_link_libraries(tificc PRIVATE  ${LIBTARGET} ${TIFF_LIBRARIES})
        target_include_directories(tificc PRIVATE ${TIFF_INCLUDE_DIR} utils/common)

        set (tool_targets ${tool_targets} tificc)
    endif()

endif()

if (BUILD_TESTS)
    enable_testing()

    # the tester requires some unexported symbols
    add_library(${LIBTARGET}-static STATIC ${SRCS} ${HDRS} )

    add_executable(
        testcms
        testbed/testcms2.c
        testbed/testcms2.h
        testbed/testplugin.c
        testbed/zoo_icc.c
    )
    target_link_libraries(testcms PRIVATE ${LIBTARGET}-static)
    target_include_directories(testcms PRIVATE src)
    
    add_test(NAME test COMMAND testcms WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/testbed")
endif()

install(TARGETS ${LIBTARGET} ${tool_targets} EXPORT LittleCms2
    RUNTIME DESTINATION "bin"
    ARCHIVE DESTINATION "lib"
    LIBRARY DESTINATION "lib"
    INCLUDES DESTINATION "include"
    PUBLIC_HEADER DESTINATION "include"
    )
    
install(EXPORT LittleCms2 DESTINATION lib/LittleCms2 FILE LittleCms2.cmake)

feature_summary(WHAT ALL)
