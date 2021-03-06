set(libvmi_src
    accessors.c
    convenience.c
    core.c
    events.c
    performance.c
    pretty_print.c
    read.c
    slat.c
    strmatch.c
    write.c
    memory.c
    arch/arch_interface.c
    arch/intel.c
    arch/amd64.c
    arch/arm_aarch32.c
    arch/arm_aarch64.c
    driver/driver_interface.c
    driver/memory_cache.c
    os/os_interface.c
)

add_library(vmi OBJECT ${libvmi_src})
# force -fPIC
set_property(TARGET vmi PROPERTY POSITION_INDEPENDENT_CODE ON)

set(VMI_PUBLIC_HEADERS
    libvmi.h
    libvmi_extra.h
    slat.h
    x86.h)

# list of external dependencies, used by libtool for the static library
set(VMI_PUBLIC_DEPS "")

# create libvmi.so
add_library (vmi_shared SHARED $<TARGET_OBJECTS:vmi>)
# performance.c uses math.h in implementation
target_link_libraries(vmi_shared PRIVATE m)
list(APPEND VMI_PUBLIC_DEPS m)
# one libvmi_extra.h function returns a GSList*
target_link_libraries(vmi_shared PUBLIC ${GLIB_LDFLAGS})
# cleanup GLIB_LDFLAGS (remove -l prefix)
string(REGEX REPLACE "-l" "" GLIB_LDFLAGS ${GLIB_LDFLAGS})
list(APPEND VMI_PUBLIC_DEPS ${GLIB_LDFLAGS})
set_target_properties(vmi_shared PROPERTIES OUTPUT_NAME "vmi")
# set soname
set_target_properties(vmi_shared PROPERTIES
    VERSION
    0.${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}
    SOVERSION 0
)
if (ENABLE_STATIC)
    # create libvmi.a
    add_library (vmi_static STATIC $<TARGET_OBJECTS:vmi>)
    set_target_properties(vmi_static PROPERTIES OUTPUT_NAME "vmi")
endif ()

# workaround CMake bug
# target_sources doesn't work with generated files in subdirectories
if (ENABLE_CONFIGFILE)
    find_package(FLEX)
    find_package(BISON)
    if (NOT FLEX_FOUND OR NOT BISON_FOUND)
        set(ENABLE_CONFIGFILE OFF CACHE "Enable config file" FORCE)
        message(WARNING "Cannot find flex or bison: config file parsing will be
        disabled")
    else ()
        add_custom_command(
            OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/scanner.c
            COMMAND ${FLEX_EXECUTABLE}
                --outfile=${CMAKE_CURRENT_BINARY_DIR}/scanner.c
                ${CMAKE_CURRENT_SOURCE_DIR}/config/lexicon.l
            COMMENT "Generating scanner.c"
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/config
        )

        add_custom_command(
            OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/parser.c
            COMMAND ${BISON_EXECUTABLE}
                --output=${CMAKE_CURRENT_BINARY_DIR}/parser.c
                --defines=${CMAKE_CURRENT_BINARY_DIR}/grammar.h
                ${CMAKE_CURRENT_SOURCE_DIR}/config/grammar.y
            COMMENT "Generating parser.c"
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/config
        )

        target_sources(vmi_shared PRIVATE
            ${CMAKE_CURRENT_BINARY_DIR}/scanner.c
            ${CMAKE_CURRENT_BINARY_DIR}/parser.c
        )
        target_include_directories(vmi_shared PRIVATE
            ${CMAKE_CURRENT_BINARY_DIR} # grammar.h
            ${CMAKE_CURRENT_SOURCE_DIR}/config # config_parser.h
        )
    endif ()
endif ()

if (ENABLE_KVM)
    find_package(JSON-C)
    find_package(Libvirt)
    if (NOT JSON-C_FOUND OR NOT Libvirt_FOUND)
        set(ENABLE_KVM OFF CACHE "Build KVM driver" FORCE)
        message(WARNING "Cannot find JSON or Libvirt: disabling KVM driver")
    else ()
        target_include_directories(vmi_shared PRIVATE ${JSON-C_INCLUDE_DIRS})
        # CMAKE_DL_LIBS -> dlopen* lib
        target_link_libraries(vmi_shared PRIVATE ${CMAKE_DL_LIBS} ${JSON-C_LIBRARIES})
        list(APPEND VMI_PUBLIC_DEPS ${CMAKE_DL_LIBS} ${JSON-C_LIBRARIES})
    endif ()
endif ()


if (ENABLE_BAREFLANK)
    find_package(JSON-C)
    if (NOT JSON-C_FOUND)
        set(ENABLE_BAREFLANK OFF CACHE "Build Bareflank driver" FORCE)
        message(WARNING "Cannot find JSON: disabling Bareflank driver")
    else ()
        target_include_directories(vmi_shared PRIVATE ${JSON-C_INCLUDE_DIRS})
        target_link_libraries(vmi_shared PRIVATE ${JSON-C_LIBRARIES})
        list(APPEND VMI_PUBLIC_DEPS ${JSON-C_LIBRARIES})
    endif ()
endif ()

add_subdirectory(driver)
add_subdirectory(os)


if (ENABLE_XEN)
    find_package(Xen REQUIRED)
    list(APPEND VMI_PUBLIC_HEADERS events.h)
    if (ENABLE_XENSTORE)
        find_package(Xenstore REQUIRED)
    endif ()
    # CMAKE_DL_LIBS -> dlopen* lib
    target_link_libraries(vmi_shared PRIVATE ${CMAKE_DL_LIBS})
    list(APPEND VMI_PUBLIC_DEPS ${CMAKE_DL_LIBS})
endif ()

if (ENABLE_WINDOWS)
    list(APPEND VMI_PUBLIC_HEADERS peparse.h)
endif ()

if (ENABLE_ADDRESS_CACHE)
    target_sources(vmi_shared PRIVATE cache.c)
endif ()

if (REKALL_PROFILES)
    find_package(JSON-C)
    if (NOT JSON-C_FOUND)
        set(REKALL_PROFILES OFF)
        message(WARNING "Cannot find JSON: disabling Rekall profiles")
    else ()
        target_sources(vmi_shared PRIVATE rekall.c)
        target_include_directories(vmi_shared PRIVATE ${JSON-C_INCLUDE_DIRS})
        target_link_libraries(vmi_shared PRIVATE ${JSON-C_LIBRARIES})
        list(APPEND VMI_PUBLIC_DEPS ${JSON-C_LIBRARIES})
    endif ()
endif ()

# install
get_property(LIB64 GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS)

if ("${LIB64}" STREQUAL "TRUE")
    set(LIBSUFFIX 64)
else()
    set(LIBSUFFIX "")
endif()

install(TARGETS vmi_shared DESTINATION lib${LIBSUFFIX})
if (ENABLE_STATIC)
    install(TARGETS vmi_static DESTINATION lib${LIBSUFFIX})
endif ()
install(FILES ${VMI_PUBLIC_HEADERS} DESTINATION include/libvmi)

list(REMOVE_DUPLICATES VMI_PUBLIC_DEPS)
set (DEPENDENCY_LIBS "")
foreach (PUB_LIB ${VMI_PUBLIC_DEPS})
    set(DEPENDENCY_LIBS "${DEPENDENCY_LIBS} -l${PUB_LIB}")
endforeach ()
# expand ${VAR}
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libtool_template.in
    ${CMAKE_CURRENT_BINARY_DIR}/libtool_template.gen)

# expand $<EXPRESSION>
file(GENERATE
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/libvmi.la
    INPUT ${CMAKE_CURRENT_BINARY_DIR}/libtool_template.gen)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libvmi.la DESTINATION lib${LIBSUFFIX})
