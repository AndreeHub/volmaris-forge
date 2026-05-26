qt_add_executable(volmaris-forge
    main.cpp
)

set_target_properties(volmaris-forge PROPERTIES
    WIN32_EXECUTABLE ON
    MACOSX_BUNDLE OFF
)

target_link_libraries(volmaris-forge
    PRIVATE
        Qt6::Core
        Qt6::Gui
        Qt6::Widgets
)

# --------------------------------------------------------------------------
# Windows: stage Qt plugins (platforms, styles, imageformats, tls,
# iconengines) next to the exe so it launches from the build directory.
#
# vcpkg's qtbase port does not ship windeployqt, so we resolve the plugins
# directory directly from Qt6_DIR (which vcpkg points at
# <triplet>/share/Qt6). Plugins live at <triplet>/Qt6/plugins.
# --------------------------------------------------------------------------
if(WIN32)
    get_filename_component(VOLMARIS_QT_PLUGINS_DIR
        "${Qt6_DIR}/../../Qt6/plugins" ABSOLUTE)

    if(NOT EXISTS "${VOLMARIS_QT_PLUGINS_DIR}/platforms/qwindows.dll")
        message(FATAL_ERROR
            "Could not locate Qt platform plugins. Looked for "
            "${VOLMARIS_QT_PLUGINS_DIR}/platforms/qwindows.dll. "
            "Qt6_DIR=${Qt6_DIR}")
    endif()

    set(_qt_plugin_subdirs platforms styles imageformats tls iconengines)
    foreach(_subdir IN LISTS _qt_plugin_subdirs)
        if(EXISTS "${VOLMARIS_QT_PLUGINS_DIR}/${_subdir}")
            add_custom_command(TARGET volmaris-forge POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy_directory
                        "${VOLMARIS_QT_PLUGINS_DIR}/${_subdir}"
                        "$<TARGET_FILE_DIR:volmaris-forge>/${_subdir}"
                COMMENT "Staging Qt plugin dir '${_subdir}' next to volmaris-forge.exe"
                VERBATIM)
        endif()
    endforeach()
endif()
