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
