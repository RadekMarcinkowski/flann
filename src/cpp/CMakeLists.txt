#include_directories(${CMAKE_SOURCE_DIR}/include algorithms util nn .)

add_definitions(-D_FLANN_VERSION=${FLANN_VERSION})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/flann/config.h.in ${CMAKE_CURRENT_SOURCE_DIR}/flann/config.h)

file(GLOB_RECURSE C_SOURCES *.cpp)
file(GLOB_RECURSE CPP_SOURCES *.cpp)
file(GLOB_RECURSE CU_SOURCES *.cu)

file(GLOB_RECURSE C_FLANN flann.cpp)
file(GLOB_RECURSE CPP_FLANN flann_cpp.cpp)

list(REMOVE_ITEM CPP_SOURCES ${C_FLANN})
list(REMOVE_ITEM C_SOURCES ${CPP_FLANN})

add_library(flann_cpp_s STATIC ${CPP_SOURCES})

# message("${CUDA_NVCC_FLAGS}")
if(CMAKE_COMPILER_IS_GNUCC)
    set_target_properties(flann_cpp_s PROPERTIES COMPILE_FLAGS -fPIC)
# 		set_target_properties(flann_cuda_s PROPERTIES COMPILE_FLAGS -fPIC)
    if (BUILD_CUDA_LIB)
    	# hack: cuda_add_libraries doesn't seem to add the -fPIC option if defined like above...
		set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};-Xcompiler;-fPIC;-arch=sm_13;--compiler-bindir=${NVCC_COMPILER_BINDIR}" )
    endif()
endif()

set_property(TARGET flann_cpp_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)

if (BUILD_CUDA_LIB)
    cuda_add_library(flann_cuda_s STATIC ${CU_SOURCES})
    set_property(TARGET flann_cuda_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_COMPILER_IS_GNUCC)
    add_library(flann_cpp SHARED "")
    set_target_properties(flann_cpp PROPERTIES LINKER_LANGUAGE CXX)
    target_link_libraries(flann_cpp -Wl,-whole-archive flann_cpp_s -Wl,-no-whole-archive)

    if (BUILD_CUDA_LIB)
	    cuda_add_library(flann_cuda SHARED "")
        set_target_properties(flann_cuda PROPERTIES LINKER_LANGUAGE CXX)
        target_link_libraries(flann_cuda -Wl,-whole-archive flann_cuda_s -Wl,-no-whole-archive)
    # 	target_link_libraries(flann_cuda cudpp_x86_64)
    endif()
else()
    add_library(flann_cpp SHARED ${CPP_SOURCES})
    if (BUILD_CUDA_LIB)
		cuda_add_library(flann_cpp SHARED ${CPP_SOURCES})
    endif()
endif()
if(TBB_FOUND AND TBB_DEBUG_FOUND)
    target_link_libraries(flann_cpp ${TBB_LIBRARIES})
endif(TBB_FOUND  AND TBB_DEBUG_FOUND)

set_target_properties(flann_cpp PROPERTIES
   VERSION ${FLANN_VERSION}
   SOVERSION ${FLANN_SOVERSION}
   DEFINE_SYMBOL FLANN_EXPORTS
) 

if (BUILD_CUDA_LIB)
    set_target_properties(flann_cuda PROPERTIES
       VERSION ${FLANN_VERSION}
       SOVERSION ${FLANN_SOVERSION}
       DEFINE_SYMBOL FLANN_EXPORTS
    ) 
endif()


#debug libraries
add_library(flann_cpp-gd SHARED ${CPP_SOURCES})
if(TBB_FOUND AND TBB_DEBUG_FOUND)
    target_link_libraries(flann_cpp-gd ${TBB_DEBUG_LIBRARIES})
endif(TBB_FOUND AND TBB_DEBUG_FOUND)
set_target_properties(flann_cpp-gd PROPERTIES 
    COMPILE_FLAGS ${CMAKE_CXX_FLAGS_DEBUG}
    DEFINE_SYMBOL FLANN_EXPORTS
)

add_library(flann_cpp_s-gd STATIC ${CPP_SOURCES})
set_target_properties(flann_cpp_s-gd PROPERTIES COMPILE_FLAGS ${CMAKE_CXX_FLAGS_DEBUG})
set_property(TARGET flann_cpp_s-gd PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)


if (BUILD_CUDA_LIB)
    add_library(flann_cuda-gd SHARED ${CPP_SOURCES})
    set_target_properties(flann_cuda-gd PROPERTIES 
        COMPILE_FLAGS ${CMAKE_CXX_FLAGS_DEBUG}
        DEFINE_SYMBOL FLANN_EXPORTS
    )

    add_library(flann_cuda_s-gd STATIC ${CPP_SOURCES})
    set_target_properties(flann_cuda_s-gd PROPERTIES COMPILE_FLAGS ${CMAKE_CXX_FLAGS_DEBUG})
    set_property(TARGET flann_cuda_s-gd PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)
endif()

if (BUILD_C_BINDINGS)
    add_library(flann_s STATIC ${C_SOURCES})
    if(CMAKE_COMPILER_IS_GNUCC)
        set_target_properties(flann_s PROPERTIES COMPILE_FLAGS -fPIC)
    endif()
    set_property(TARGET flann_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)

    if(CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_COMPILER_IS_GNUCC)
        add_library(flann SHARED "")
        set_target_properties(flann PROPERTIES LINKER_LANGUAGE CXX)
        target_link_libraries(flann -Wl,-whole-archive flann_s -Wl,-no-whole-archive)
    else()
        add_library(flann SHARED ${C_SOURCES})
    endif()

    set_target_properties(flann PROPERTIES
       VERSION ${FLANN_VERSION}
       SOVERSION ${FLANN_SOVERSION}
       DEFINE_SYMBOL FLANN_EXPORTS
    ) 
endif()


if(WIN32)
if (BUILD_C_BINDINGS)
    install (
        TARGETS flann
        RUNTIME DESTINATION share/flann/matlab
    )
endif()
endif(WIN32)


install (
    TARGETS flann_cpp flann_cpp_s flann_cpp-gd flann_cpp_s-gd
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION ${FLANN_LIB_INSTALL_DIR}
    ARCHIVE DESTINATION ${FLANN_LIB_INSTALL_DIR}
)
 
if (BUILD_CUDA_LIB)
    install (
        TARGETS flann_cuda flann_cuda_s flann_cuda-gd flann_cuda_s-gd
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION ${FLANN_LIB_INSTALL_DIR}
        ARCHIVE DESTINATION ${FLANN_LIB_INSTALL_DIR}
    )
endif()

if (BUILD_C_BINDINGS)
    install (
        TARGETS flann flann_s
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION ${FLANN_LIB_INSTALL_DIR}
        ARCHIVE DESTINATION ${FLANN_LIB_INSTALL_DIR}
    )
endif()

install (
    DIRECTORY flann
    DESTINATION include
    FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
)
