function(add_startup_object name)
  cmake_parse_arguments(
    "ADD_STARTUP_OBJECT"
    "ALIAS" # Option argument
    "SRC"   # Single value arguments
    "DEPENDS;COMPILE_OPTIONS" # Multi value arguments
    ${ARGN}
  )

  get_fq_target_name(${name} fq_target_name)
  get_fq_deps_list(fq_deps_list ${ADD_STARTUP_OBJECT_DEPENDS})
  if(ADD_STARTUP_OBJECT_ALIAS)
    list(LENGTH ADD_STARTUP_OBJECT_DEPENDS deps_size)
    if(NOT (${deps_size} EQUAL "1"))
      message(FATAL_ERROR "A startup object alias should have exactly one dependency.")
    endif()
    list(GET ADD_STARTUP_OBJECT_DEPENDS 0 dep)
    get_fq_dep_name(fq_dep_name ${dep})

    add_custom_target(${fq_target_name})
    add_dependencies(${fq_target_name} ${fq_dep_name})
    get_target_property(startup_object ${fq_dep_name} STARTUP_OBJECT)
    set_target_properties(
      ${fq_target_name}
      PROPERTIES
        "TARGET_TYPE" "${OBJECT_LIBRARY_TARGET_TYPE}"
        "STARTUP_OBJECT" "${startup_object}"
        "OBJECT_FILES" ""
        "DEPS" "${fq_dep_name}"
    )
    return()
  endif()

  add_object_library(
    ${name}.__objects__
    SRCS ${ADD_STARTUP_OBJECT_SRC}
    DEPENDS ${ADD_STARTUP_OBJECT_DEPENDS}
    COMPILE_OPTIONS ${ADD_STARTUP_OBJECT_COMPILE_OPTIONS}
  )

  set(objfile ${LIBC_BUILD_DIR}/lib/${name}.o)
  add_custom_command(
    OUTPUT ${objfile}
    COMMAND cp $<TARGET_OBJECTS:${fq_target_name}.__objects__> ${objfile}
    DEPENDS $<TARGET_OBJECTS:${fq_target_name}.__objects__>
  )
  add_custom_target(
    ${fq_target_name}
    DEPENDS ${objfile}
  )
  set_target_properties(
    ${fq_target_name}
    PROPERTIES
      "TARGET_TYPE" "${OBJECT_LIBRARY_TARGET_TYPE}"
      "STARTUP_OBJECT" "${objfile}"
      "OBJECT_FILES" ""
      "DEPS" "${fq_target_name}.__objects__"
  )
endfunction()

if(NOT (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${LIBC_TARGET_ARCHITECTURE}))
  message(STATUS "Skipping startup for target architecture ${LIBC_TARGET_ARCHITECTURE}")
  return()
endif()

add_subdirectory(${LIBC_TARGET_ARCHITECTURE})

add_startup_object(
  crt1
  ALIAS
  DEPENDS
    .${LIBC_TARGET_ARCHITECTURE}.crt1
)

add_startup_object(
  crti
  SRC
    crti.cpp
)

add_startup_object(
  crtn
  SRC
    crtn.cpp
)

add_custom_target(libc-startup)
set(startup_components crt1 crti crtn)
foreach(target IN LISTS startup_components)
  set(fq_target_name libc.startup.linux.${target})
  add_dependencies(libc-startup ${fq_target_name})
  get_target_property(startup_object ${fq_target_name} STARTUP_OBJECT)
  install(FILES ${startup_object}
          DESTINATION ${CMAKE_INSTALL_LIBDIR}
          COMPONENT libc)
endforeach()
