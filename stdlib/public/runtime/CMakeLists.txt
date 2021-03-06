set(swift_runtime_compile_flags ${SWIFT_RUNTIME_CORE_CXX_FLAGS})
set(swift_runtime_linker_flags ${SWIFT_RUNTIME_CORE_LINK_FLAGS})

if(SWIFT_RUNTIME_CLOBBER_FREED_OBJECTS)
  list(APPEND swift_runtime_compile_flags
      "-DSWIFT_RUNTIME_CLOBBER_FREED_OBJECTS=1")
endif()

if(SWIFT_RUNTIME_CRASH_REPORTER_CLIENT)
  list(APPEND swift_runtime_compile_flags
      "-DSWIFT_HAVE_CRASHREPORTERCLIENT=1")
endif()

set(swift_runtime_leaks_sources)
if(SWIFT_RUNTIME_ENABLE_LEAK_CHECKER)
  list(APPEND swift_runtime_compile_flags
       "-DSWIFT_RUNTIME_ENABLE_LEAK_CHECKER=1")
  set(swift_runtime_leaks_sources Leaks.mm)
endif()

set(swift_runtime_objc_sources
    ErrorObject.mm
    SwiftObject.mm
    SwiftValue.mm
    ReflectionMirror.mm
    ObjCRuntimeGetImageNameFromClass.mm
    "${SWIFT_SOURCE_DIR}/lib/Demangling/OldRemangler.cpp"
    "${SWIFT_SOURCE_DIR}/lib/Demangling/Remangler.cpp"
    )

set(swift_runtime_sources
    AnyHashableSupport.cpp
    Array.cpp
    BackDeployment.cpp
    Casting.cpp
    CompatibilityOverride.cpp
    CygwinPort.cpp
    Demangle.cpp
    Enum.cpp
    ErrorObjectCommon.cpp
    ErrorObjectConstants.cpp
    ErrorObjectNative.cpp
    Errors.cpp
    ErrorDefaultImpls.cpp
    Exclusivity.cpp
    ExistentialContainer.cpp
    Heap.cpp
    HeapObject.cpp
    ImageInspectionMachO.cpp
    ImageInspectionELF.cpp
    ImageInspectionCOFF.cpp
    KeyPaths.cpp
    KnownMetadata.cpp
    LLVMSupport.cpp
    Metadata.cpp
    MetadataLookup.cpp
    MutexPThread.cpp
    MutexWin32.cpp
    MutexWASI.cpp
    Numeric.cpp
    Once.cpp
    Portability.cpp
    ProtocolConformance.cpp
    RefCount.cpp
    RuntimeInvocationsTracking.cpp
    SwiftDtoa.cpp
    "${SWIFT_SOURCE_DIR}/lib/Demangling/OldDemangler.cpp"
    "${SWIFT_SOURCE_DIR}/lib/Demangling/Demangler.cpp"
    "${SWIFT_SOURCE_DIR}/lib/Demangling/NodePrinter.cpp"
    "${SWIFT_SOURCE_DIR}/lib/Demangling/Context.cpp"
    "${SWIFT_SOURCE_DIR}/lib/Demangling/ManglingUtils.cpp"
    "${SWIFT_SOURCE_DIR}/lib/Demangling/Punycode.cpp")

# When we're building with assertions, include the demangle node dumper to aid
# in debugging.
if (LLVM_ENABLE_ASSERTIONS)
  list(APPEND swift_runtime_sources "${SWIFT_SOURCE_DIR}/lib/Demangling/NodeDumper.cpp")
endif(LLVM_ENABLE_ASSERTIONS)

# Acknowledge that the following sources are known.
set(LLVM_OPTIONAL_SOURCES
    SwiftRT-COFF.cpp
    SwiftRT-ELF.cpp
    SwiftRT-WASM.cpp
    ${swift_runtime_sources}
    ${swift_runtime_objc_sources}
    ${swift_runtime_leaks_sources})

set(swift_runtime_library_compile_flags ${swift_runtime_compile_flags})
list(APPEND swift_runtime_library_compile_flags -DswiftCore_EXPORTS)
list(APPEND swift_runtime_library_compile_flags -I${SWIFT_SOURCE_DIR}/include)

set(static_binary_lnk_file_list)
set(static_binary_dependencies_list)
macro(add_image_inspection_shared sdk primary_arch inspection_file linkfile_src)
  if(${inspection_file} IN_LIST swift_runtime_sources)
    list(REMOVE_ITEM swift_runtime_sources ${inspection_file})
  endif()
  string(TOLOWER "${sdk}" lowercase_sdk)

  # These two libraries are only used with the static swiftcore
  add_swift_target_library(swiftImageInspectionShared STATIC
    ${inspection_file}
    C_COMPILE_FLAGS ${swift_runtime_library_compile_flags}
    LINK_FLAGS ${swift_runtime_linker_flags}
    SWIFT_COMPILE_FLAGS ${SWIFT_STANDARD_LIBRARY_SWIFT_FLAGS}
    TARGET_SDKS ${sdk}
    INSTALL_IN_COMPONENT stdlib)

  foreach(arch IN LISTS SWIFT_SDK_${sdk}_ARCHITECTURES)
    set(FragileSupportLibrary swiftImageInspectionShared-${SWIFT_SDK_${sdk}_LIB_SUBDIR}-${arch})
    set(LibraryLocation ${SWIFTSTATICLIB_DIR}/${lowercase_sdk}/${arch})

    add_custom_command_target(swift_image_inspection_${lowercase_sdk}_${arch}_static
      COMMAND
        "${CMAKE_COMMAND}" -E copy $<TARGET_FILE:${FragileSupportLibrary}> ${LibraryLocation}
      OUTPUT
        "${LibraryLocation}/${CMAKE_STATIC_LIBRARY_PREFIX}swiftImageInspectionShared${CMAKE_STATIC_LIBRARY_SUFFIX}"
      DEPENDS
         ${FragileSupportLibrary})

    list(APPEND static_binary_dependencies_list ${swift_image_inspection_${lowercase_sdk}_${arch}_static})
    add_dependencies(stdlib ${FragileSupportLibrary})
    swift_install_in_component(FILES $<TARGET_FILE:${FragileSupportLibrary}>
                               DESTINATION "lib/swift_static/${lowercase_sdk}/${arch}"
                               COMPONENT stdlib)
  endforeach()

  if(NOT "${primary_arch}" STREQUAL "")
    set(FragileSupportLibraryPrimary swiftImageInspectionShared-${SWIFT_SDK_${sdk}_LIB_SUBDIR}-${primary_arch})
    set(LibraryLocationPrimary ${SWIFTSTATICLIB_DIR}/${lowercase_sdk})
    add_custom_command_target(swift_image_inspection_static_${lowercase_sdk}_primary_arch
        COMMAND
           "${CMAKE_COMMAND}" -E copy $<TARGET_FILE:${FragileSupportLibraryPrimary}> ${LibraryLocationPrimary}
        OUTPUT
        "${LibraryLocationPrimary}/${CMAKE_STATIC_LIBRARY_PREFIX}swiftImageInspectionShared${CMAKE_STATIC_LIBRARY_SUFFIX}"
        DEPENDS
           ${FragileSupportLibraryPrimary})
       list(APPEND static_binary_dependencies_list ${swift_image_inspection_static_${lowercase_sdk}_primary_arch})
    add_dependencies(stdlib ${FragileSupportLibraryPrimary})
    swift_install_in_component(FILES $<TARGET_FILE:${FragileSupportLibraryPrimary}>
                               DESTINATION "lib/swift_static/${lowercase_sdk}"
                               COMPONENT stdlib)
  endif()

  set(linkfile "${lowercase_sdk}/static-executable-args.lnk")
  add_custom_command_target(swift_static_binary_${sdk}_args
    COMMAND
      "${CMAKE_COMMAND}" -E copy
      "${linkfile_src}"
      "${SWIFTSTATICLIB_DIR}/${linkfile}"
    OUTPUT
      "${SWIFTSTATICLIB_DIR}/${linkfile}"
    DEPENDS
      "${linkfile_src}")

  list(APPEND static_binary_lnk_file_list ${swift_static_binary_${sdk}_args})
  swift_install_in_component(FILES "${SWIFTSTATICLIB_DIR}/${linkfile}"
                             DESTINATION "lib/swift_static/${lowercase_sdk}"
                             COMPONENT stdlib)

  add_swift_target_library(swiftImageInspectionSharedObject OBJECT_LIBRARY
    ${inspection_file}
    C_COMPILE_FLAGS ${swift_runtime_library_compile_flags}
    LINK_FLAGS ${swift_runtime_linker_flags}
    SWIFT_COMPILE_FLAGS ${SWIFT_STANDARD_LIBRARY_SWIFT_FLAGS}
    TARGET_SDKS ${sdk}
    INSTALL_IN_COMPONENT never_install)
endmacro()

set(is_image_inspection_required)
foreach(sdk IN LISTS SWIFT_SDKS)
  set(image_inspection_shared_sdk)
  set(primary_arch)
  set(image_inspection_shared_file)
  set(linkfile_src)

  if("${SWIFT_SDK_${sdk}_OBJECT_FORMAT}" STREQUAL "ELF")
    list(APPEND ELFISH_SDKS ${sdk})
    set(linkfile_src "${SWIFT_SOURCE_DIR}/utils/static-executable-args.lnk")
  elseif("${SWIFT_SDK_${sdk}_OBJECT_FORMAT}" STREQUAL "WASM")
    set(linkfile_src "${SWIFT_SOURCE_DIR}/utils/webassembly/static-executable-args.lnk")
  endif()

  if(SWIFT_BUILD_STATIC_STDLIB AND "${sdk}" STREQUAL "LINUX")
    set(image_inspection_shared_sdk "${sdk}")
    set(image_inspection_shared_file ImageInspectionELF.cpp)
  elseif(SWIFT_BUILD_STATIC_STDLIB AND "${sdk}" STREQUAL "WASI")
    set(image_inspection_shared_sdk "${sdk}")
    set(image_inspection_shared_file ImageInspectionELF.cpp)
    # Set default arch
    set(primary_arch "wasm32")
  endif()

  if("${sdk}" STREQUAL "${SWIFT_PRIMARY_VARIANT_SDK}")
    set(primary_arch ${SWIFT_PRIMARY_VARIANT_ARCH})
  endif()

  if(NOT "${image_inspection_shared_sdk}" STREQUAL "" AND NOT "${primary_arch}" STREQUAL "")
    set(is_image_inspection_required TRUE)
    add_image_inspection_shared(${image_inspection_shared_sdk} ${primary_arch} ${image_inspection_shared_file} ${linkfile_src})
  endif()
endforeach()

if(is_image_inspection_required)
  add_custom_target(static_binary_magic ALL DEPENDS ${static_binary_lnk_file_list} ${static_binary_dependencies_list})
  add_dependencies(stdlib static_binary_magic)
endif()

add_swift_target_library(swiftRuntime OBJECT_LIBRARY
  ${swift_runtime_sources}
  ${swift_runtime_objc_sources}
  ${swift_runtime_leaks_sources}
  C_COMPILE_FLAGS ${swift_runtime_library_compile_flags}
  LINK_FLAGS ${swift_runtime_linker_flags}
  SWIFT_COMPILE_FLAGS ${SWIFT_STANDARD_LIBRARY_SWIFT_FLAGS}
  INSTALL_IN_COMPONENT never_install)

set(ELFISH_SDKS)
set(COFF_SDKS)
set(WASM_SDKS)
foreach(sdk ${SWIFT_CONFIGURED_SDKS})
  if("${SWIFT_SDK_${sdk}_OBJECT_FORMAT}" STREQUAL "ELF")
    list(APPEND ELFISH_SDKS ${sdk})
  elseif("${SWIFT_SDK_${sdk}_OBJECT_FORMAT}" STREQUAL "COFF")
    list(APPEND COFF_SDKS ${sdk})
  elseif("${SWIFT_SDK_${sdk}_OBJECT_FORMAT}" STREQUAL "WASM")
    list(APPEND WASM_SDKS ${sdk})
  endif()
endforeach()

add_swift_target_library(swiftImageRegistrationObjectELF
                  OBJECT_LIBRARY IS_STDLIB IS_STDLIB_CORE
                  SwiftRT-ELF.cpp
                  C_COMPILE_FLAGS ${SWIFT_RUNTIME_CORE_CXX_FLAGS}
                  LINK_FLAGS ${SWIFT_RUNTIME_CORE_LINK_FLAGS}
                  TARGET_SDKS ${ELFISH_SDKS}
                  SWIFT_COMPILE_FLAGS ${SWIFT_STANDARD_LIBRARY_SWIFT_FLAGS}
                  INSTALL_IN_COMPONENT none)

add_swift_target_library(swiftImageRegistrationObjectWASM
                  OBJECT_LIBRARY IS_STDLIB IS_STDLIB_CORE
                  SwiftRT-WASM.cpp
                  C_COMPILE_FLAGS ${SWIFT_RUNTIME_CORE_CXX_FLAGS}
                  LINK_FLAGS ${SWIFT_RUNTIME_CORE_LINK_FLAGS}
                  TARGET_SDKS ${WASM_SDKS}
                  SWIFT_COMPILE_FLAGS ${SWIFT_STANDARD_LIBRARY_SWIFT_FLAGS}
                  INSTALL_IN_COMPONENT none)

# FIXME(compnerd) this should be compiled twice, once for static and once for
# shared.  The static version should be used for building the standard library.
add_swift_target_library(swiftImageRegistrationObjectCOFF
                  OBJECT_LIBRARY IS_STDLIB IS_STDLIB_CORE
                  SwiftRT-COFF.cpp
                  C_COMPILE_FLAGS ${SWIFT_RUNTIME_CORE_CXX_FLAGS}
                  LINK_FLAGS ${SWIFT_RUNTIME_CORE_LINK_FLAGS}
                  TARGET_SDKS ${COFF_SDKS}
                  SWIFT_COMPILE_FLAGS ${SWIFT_STANDARD_LIBRARY_SWIFT_FLAGS}
                  INSTALL_IN_COMPONENT none)

foreach(sdk ${SWIFT_CONFIGURED_SDKS})
  foreach(arch ${SWIFT_SDK_${sdk}_ARCHITECTURES})
    set(arch_subdir "${SWIFT_SDK_${sdk}_LIB_SUBDIR}/${arch}")
    set(arch_suffix "${SWIFT_SDK_${sdk}_LIB_SUBDIR}-${arch}")

    if("${SWIFT_SDK_${sdk}_OBJECT_FORMAT}" STREQUAL "ELF" OR
       "${SWIFT_SDK_${sdk}_OBJECT_FORMAT}" STREQUAL "COFF" OR
       "${SWIFT_SDK_${sdk}_OBJECT_FORMAT}" STREQUAL "WASM")
      # TODO(compnerd) switch to the generator expression when cmake is upgraded
      # to a version which supports it.
      # set(swiftrtObject "$<TARGET_OBJECTS:swiftImageRegistrationObject${SWIFT_SDK_${sdk}_OBJECT_FORMAT}-${arch_suffix}>")
      set(swiftrtObject ${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/swiftImageRegistrationObject${SWIFT_SDK_${sdk}_OBJECT_FORMAT}-${arch_suffix}.dir/SwiftRT-${SWIFT_SDK_${sdk}_OBJECT_FORMAT}.cpp${CMAKE_C_OUTPUT_EXTENSION})

      if(sdk STREQUAL WINDOWS)
        set(extension .obj)
      else()
        set(extension .o)
      endif()

      set(shared_runtime_registrar "${SWIFTLIB_DIR}/${arch_subdir}/swiftrt${extension}")
      set(static_runtime_registrar "${SWIFTSTATICLIB_DIR}/${arch_subdir}/swiftrt${extension}")

      add_custom_command_target(swiftImageRegistration-${arch_suffix}
                                COMMAND
                                  "${CMAKE_COMMAND}" -E copy "${swiftrtObject}" "${shared_runtime_registrar}"
                                COMMAND
                                  "${CMAKE_COMMAND}" -E copy "${swiftrtObject}" "${static_runtime_registrar}"
                                OUTPUT
                                  "${shared_runtime_registrar}"
                                  "${static_runtime_registrar}"
                                DEPENDS
                                  "${swiftrtObject}")
      if(SWIFT_BUILD_DYNAMIC_STDLIB)
        swift_install_in_component(FILES
                                     "${shared_runtime_registrar}"
                                   DESTINATION
                                     "lib/swift/${arch_subdir}"
                                   COMPONENT
                                     stdlib)
      endif()
      if(SWIFT_BUILD_STATIC_STDLIB)
        swift_install_in_component(FILES
                                     "${static_runtime_registrar}"
                                   DESTINATION
                                     "lib/swift_static/${arch_subdir}"
                                   COMPONENT
                                     stdlib)
      endif()

      add_dependencies(swift-stdlib-${arch_suffix} ${swiftImageRegistration-${arch_suffix}})

      add_custom_target(swiftImageRegistration-${arch_suffix}
                        ALL DEPENDS
                          ${swiftImageRegistration-${arch_suffix}})
      add_dependencies(stdlib swift-stdlib-${arch_suffix} swiftImageRegistration-${arch_suffix})
    endif()
  endforeach()
endforeach()

