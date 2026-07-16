# Compute the LLVM libraries to include.
llvm_map_components_to_libnames(llvm_libraries "all")
# tblgen libraries are only useful for building. It serves no purpose in the
# dylib.
message(FATAL_ERROR "llvm libs: ${llvm_libraries}")
list(REMOVE_ITEM llvm_libraries "LLVMTableGen")
list(REMOVE_DUPLICATES llvm_libraries)

# Compute the clang libraries to include.
# CLANG_EXPORTED_TARGETS includes non-library targets. Filter them out.
set(clang_libraries)
foreach(t ${CLANG_EXPORTED_TARGETS})
  if (TARGET ${t})
    get_target_property(_type ${t} TYPE)
    if(_type STREQUAL "STATIC_LIBRARY" OR _type STREQUAL "OBJECT_LIBRARY")
      list(APPEND clang_libraries ${t})
    endif()
  endif()
endforeach()

# Compute the swift libraries to include.
set(_stdlib_suffixes)
foreach(sdk ${SWIFT_CONFIGURED_SDKS})
  foreach(arch ${SWIFT_SDK_${sdk}_ARCHITECTURES})
    list(APPEND _stdlib_suffixes "-${SWIFT_SDK_${sdk}_LIB_SUBDIR}-${arch}")
  endforeach()
endforeach()
get_property(swift_exports GLOBAL PROPERTY SWIFT_EXPORTS)
get_property(swift_buildtree_exports GLOBAL PROPERTY SWIFT_BUILDTREE_EXPORTS)

set(swift_libraries)
foreach(t ${swift_exports} ${swift_buildtree_exports})
  if (NOT TARGET ${t})
    continue()
  endif()

  # SKIP STDLIB TARGETS
  set(should_skip FALSE)
  foreach(variant ${_stdlib_suffixes})
    if (t MATCHES "${variant}$")
      set(should_skip TRUE)
      break()
    endif()
  endforeach()

  # HACK: If it starts with an underscore, make sure it starts with _Compiler!
  if ("${t}" MATCHES "^_" AND NOT "${t}" MATCHES "^_Compiler")
    set(should_skip TRUE)
  endif()
      
  if (should_skip)
    continue()
  endif()


  # SKIP NON-LIBRARIES
  get_target_property(_type ${t} TYPE)
  if (_type STREQUAL "STATIC_LIBRARY" OR _type STREQUAL "OBJECT_LIBRARY")
    list(APPEND swift_libraries ${t})
  endif()


endforeach()
# The stub is an implementation detail, we actually care about swiftCompilerModules.
list(REMOVE_ITEM swift_libraries "swiftCompilerStub")
list(REMOVE_DUPLICATES swift_libraries)

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/llvm.txt" "${llvm_libraries}")
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/clang.txt" "${clang_libraries}")
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/swift.txt" "${swift_libraries}")

add_swift_host_library(SwiftCompilerShared SHARED
  libswiftcompiler.cpp
)

# Can't use -all_load because of auto-link. Several of the swift libraries have
# LC_LINKER_OPTIONS (e.g. -lswiftASTGen) which forces the linker to load a
# static archive an additional time. Because the link arguments will pass a
# full path to the archive and the auto-link mechanism uses a relative path,
# the linker cannot reason that they are the same and loads everything twice.

#set(all_libraries ${llvm_libraries} ${clang_libraries} ${swift_libraries})
#set(all_libraries)
#foreach(lib ${llvm_libraries} ${clang_libraries} ${swift_libraries})
#  get_target_property(_type ${lib} TYPE)
#  if (_type STREQUAL "STATIC_LIBRARY")
#    list(APPEND all_libraries "$<TARGET_LINKER_FILE:${lib}>")
#  else()
#    list(APPEND all_libraries "$<TARGET_OBJECTS:${lib}>")
#  endif()
#  add_dependencies(SwiftCompilerShared ${lib})
#endforeach()
#set(link_args -Wl,-all_load ${all_libraries})

set(link_args)
foreach(lib ${llvm_libraries} ${clang_libraries})
  list(APPEND link_args ${lib} -Wl,-force_load,$<TARGET_LINKER_FILE:${lib}>)
endforeach()

foreach(lib ${swift_libraries})
  get_target_property(_is_pure_swift ${lib} PURE_SWIFT_LIB)
  get_target_property(_type ${lib} TYPE)
  if (${_is_pure_swift})
    list(APPEND link_args ${lib})
  else()
    if (_type STREQUAL "STATIC_LIBRARY")
      list(APPEND link_args ${lib} -Wl,-force_load,$<TARGET_LINKER_FILE:${lib}>)
    else()
      list(APPEND link_args ${lib})
    endif()
  endif()
endforeach()

target_link_libraries(SwiftCompilerShared PRIVATE ${link_args})
