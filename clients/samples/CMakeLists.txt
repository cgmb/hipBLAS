# ########################################################################
# Copyright 2016-2020 Advanced Micro Devices, Inc.
# ########################################################################

set( hipblas_samples_common ../common/utility.cpp )

add_executable( example-sscal example_sscal.cpp ${hipblas_samples_common} )
add_executable( example-sgemm example_sgemm.cpp ${hipblas_samples_common} )
add_executable( example-sgemm-strided-batched example_sgemm_strided_batched.cpp ${hipblas_samples_common} )
add_executable( example-c example-c.c ${hipblas_samples_common} )
#add_executable( example-sscal-fortran example_sscal_fortran.f90 )

# We test for C99 compatibility in the example-c.c test
set_source_files_properties(example-c.c PROPERTIES LANGUAGE CXX)
set_source_files_properties(example-c.c PROPERTIES COMPILE_FLAGS "-xc -std=c99")

# We test for C++11 compatibility in one of the samples
set_source_files_properties(example_sgemm_strided_batched.cpp PROPERTIES COMPILE_FLAGS "-std=c++11")

if( NOT TARGET hipblas )
  find_package( hipblas CONFIG PATHS ${ROCM_PATH}/hipblas )
  if( NOT hipblas_FOUND )
    message( FATAL_ERROR "hipBLAS is a required dependency and is not found; try adding rocblas path to CMAKE_PREFIX_PATH")
  endif( )
endif( )

foreach( exe example-sscal;example-sgemm;example-sgemm-strided-batched;example-c )

  target_link_libraries( ${exe} PRIVATE roc::hipblas )

  # External header includes included as SYSTEM files
  target_include_directories( ${exe}
    SYSTEM PRIVATE
      $<BUILD_INTERFACE:${HIP_INCLUDE_DIRS}>
      ${ROCM_PATH}/hsa/include
  )
  set_target_properties(${exe} PROPERTIES LINKER_LANGUAGE CXX)
  set_target_properties( ${exe} PROPERTIES DEBUG_POSTFIX "-d" CXX_EXTENSIONS NO )
  set_target_properties( ${exe} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging" )
  target_include_directories( ${exe} PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include> )

  if( NOT CUDA_FOUND )
    target_compile_definitions( ${exe} PRIVATE __HIP_PLATFORM_HCC__ )

  # Remove following when hcc is fixed; hcc emits following spurious warning
  # "clang-5.0: warning: argument unused during compilation: '-isystem /opt/rocm/include'"
  target_compile_options( ${exe} PRIVATE -mf16c)

  if( CUSTOM_TARGET )
    target_link_libraries( ${exe} PRIVATE hip::${CUSTOM_TARGET} hip::host )
  else( )
    if( LIBAMDHIP64_LIBRARY )
      target_link_libraries( ${exe} PRIVATE hip::amdhip64 hip::host )
    else( )
      get_target_property( HIP_HCC_LOCATION hip::hip_hcc IMPORTED_LOCATION_RELEASE )
      target_link_libraries( ${exe} PRIVATE ${HIP_HCC_LOCATION} )
    endif( )
  endif( )

  if( CMAKE_COMPILER_IS_GNUCXX )
    # GCC needs specific flag to turn on f16c intrinsics
    target_compile_options( ${exe} PRIVATE -mf16c )
  endif()


  else( )
    target_compile_definitions( ${exe} PRIVATE __HIP_PLATFORM_NVCC__ )

    target_include_directories( ${exe}
      PRIVATE
        $<BUILD_INTERFACE:${CUDA_INCLUDE_DIRS}>
    )

    target_link_libraries( ${exe} PRIVATE ${CUDA_LIBRARIES} )
  endif( )

endforeach( )
