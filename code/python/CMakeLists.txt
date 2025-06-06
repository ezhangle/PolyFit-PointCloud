cmake_minimum_required(VERSION 3.12)
project(PyPolyFit)

set(CMAKE_CXX_STANDARD 11)  # If you use pybind11::overload_cast<...>, then it requires compiling in C++14 mode

pybind11_add_module(${PROJECT_NAME} MODULE bindings/polyfit.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE method)
target_include_directories(${PROJECT_NAME} PRIVATE ${POLYFIT_INCLUDE_DIR})
target_compile_definitions(${PROJECT_NAME} PRIVATE PYBIND11_SIMPLE_GIL_SAFE_ABI) # to use the Python Stable ABI

set(POLYFIT_MODULE_NAME "PyPolyFit") # Sets the base name of the output file
target_compile_definitions(${PROJECT_NAME} PUBLIC "PyPolyFit_NAME=${POLYFIT_MODULE_NAME}")

# Platform-specific suffix for Python extensions
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(POLYFIT_MODULE_SUFFIX ".pyd")
    set_target_properties(${PROJECT_NAME} PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}) # put it together with exes/dlls
else()
    set(POLYFIT_MODULE_SUFFIX ".so")
endif()
# Set the target properties with the computed suffix
set_target_properties(${PROJECT_NAME} PROPERTIES
        OUTPUT_NAME ${POLYFIT_MODULE_NAME}  # Sets the base name of the output file
        SUFFIX "${POLYFIT_MODULE_SUFFIX}"   # Use the computed suffix
        FOLDER "python"
        )



# ----------------------------------------------------------------------------------------------------------------------
# The code below is actually not necessary for generating bindings.
# They are here to make it easier for the installation of the generated Python module.
# ----------------------------------------------------------------------------------------------------------------------

# Post-build command to copy the compiled module and its dependencies to the Python package directory.
# This function currently only works on Windows. On macOS and Linux the rpaths are not correctly resolved.
#function(copy_runtime_dependencies target destination)
#    if (WIN32)  # MSVC or MINGW
#        add_custom_command(TARGET ${target} POST_BUILD
#                COMMAND ${CMAKE_COMMAND} -E make_directory ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${target}> ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND ${CMAKE_COMMAND} -E echo "Copying runtime DLLs of ${target} to ${destination}..."
#                # -----------------------------------------------------------------------------------------------
#                # Liangliang: both "dumpbin /DEPENDENTS" and "$<TARGET_RUNTIME_DLLS:${target}>" only resolve direct dependencies.
#                # COMMAND dumpbin /DEPENDENTS $<TARGET_FILE:${target}> > ${destination}/runtime_dependencies.txt
#                # COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_RUNTIME_DLLS:${target}> ${destination}
#                # ----------
#                # Using a Python script (deploy.py) to resolve transitive dependencies.
#                # This script should also work for macOS and Linux!!!
#                COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_LIST_DIR}/deploy.py $<TARGET_FILE:${target}>  ${destination}
#                # -----------------------------------------------------------------------------------------------
#                # Copy the Python runtime DLL
#                COMMAND ${CMAKE_COMMAND} -E echo "Copying Python runtime DLL ${PYTHON_RUNTIME_DLL} to ${destination}..."
#                COMMAND ${CMAKE_COMMAND} -E copy_if_different ${PYTHON_RUNTIME_DLL} ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND_EXPAND_LISTS
#        )
#    elseif (APPLE) # macOS
#        add_custom_command(TARGET ${target} POST_BUILD
#                COMMAND ${CMAKE_COMMAND} -E make_directory ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${target}> ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND ${CMAKE_COMMAND} -E echo "Copying runtime dependencies of ${target} to ${destination}..."
#                # Liangliang: It took me so much time to figure out how to find and copy the dependencies using `add_custom_command`.
#                # Write the dependencies of the target to a file:
#                #   (a) otool -L $<TARGET_FILE:${target}>: Lists the dynamic library dependencies of the target file (regardless of file existence).
#                #   (b) awk '/@rpath|\\/|\\// {print $$1}': Extracts the first column (the library paths) from the output of otool.
#                #   (c) sed 's|@rpath|$<TARGET_FILE_DIR:${target}>|g': Replaces @rpath with the target file directory.
#                #   (d) grep -v "^$<TARGET_FILE:${target}>$": Removes the line containing the target file. That is the heading line of the otool
#                #       output, e.g., "/.../Release/lib/PyPolyFit.so:", which is not shown when using the command in terminal.
#                COMMAND otool -L $<TARGET_FILE:${target}> | awk '/@rpath|\\/|\\// {print $$1}' | sed 's|@rpath|$<TARGET_FILE_DIR:${target}>|g' | grep -v "^$<TARGET_FILE:${target}>$" > ${destination}/runtime_dependencies.txt
#                # Copy the dependencies of the target to the destination directory:
#                #   (e) xargs -I {} cp -v {} ${destination}: Copies each library file to the destination directory. If does not exist, it will show e.g., "cp: /usr/lib/libc++.1.dylib: No such file or directory".
#                #   (f) || true: Ignores the error during the copy process. This ensures that the command does not cause the build to fail.
#                COMMAND otool -L $<TARGET_FILE:${target}> | awk '/@rpath|\\/|\\// {print $$1}' | sed 's|@rpath|$<TARGET_FILE_DIR:${target}>|g' | grep -v "^$<TARGET_FILE:${target}>$" | xargs -I {} cp -v {} ${destination} || true
#                #   (g) xargs -I {} sh -c '[ -e "{}" ] && cp -v "{}" ${destination}': Ensures only existing files are copied to the destination directory. Thus it avoids the "No such file or directory" message.
#                #COMMAND otool -L $<TARGET_FILE:${target}> | awk '/@rpath|\\/|\\// {print $$1}' | sed 's|@rpath|$<TARGET_FILE_DIR:${target}>|g' | xargs -I {} sh -c '[ -e "{}" ] && cp -v "{}" ${destination}' || true
#                # -----------------------------------------------------------------------------------------------
#                # Copy the Python runtime library
#                COMMAND ${CMAKE_COMMAND} -E echo "Copying Python runtime library ${PYTHON_LIBRARIES} to ${destination}..."
#                COMMAND ${CMAKE_COMMAND} -E copy_if_different ${PYTHON_LIBRARIES} ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND_EXPAND_LISTS
#        )
#    elseif (UNIX) # Linux
#        add_custom_command(TARGET ${target} POST_BUILD
#                COMMAND ${CMAKE_COMMAND} -E make_directory ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${target}> ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND ${CMAKE_COMMAND} -E echo "Copying runtime dependencies of ${target} to ${destination}..."
#                # Liangliang: It took me so much time to figure out how to find and copy the dependencies using `add_custom_command`.
#                COMMAND ldd $<TARGET_FILE:${target}> | awk '/=>/ {print $$3}' > ${destination}/runtime_dependencies.txt
#                COMMAND ldd $<TARGET_FILE:${target}> | grep '=> /' | awk '{print $$3}' | xargs -I {} cp -v {} ${destination}
#                # -----------------------------------------------------------------------------------------------
#                # Copy the Python runtime library
#                COMMAND ${CMAKE_COMMAND} -E echo "Copying Python runtime library ${PYTHON_LIBRARIES} to ${destination}..."
#                COMMAND ${CMAKE_COMMAND} -E copy_if_different ${PYTHON_LIBRARIES} ${destination}
#                # -----------------------------------------------------------------------------------------------
#                COMMAND_EXPAND_LISTS
#        )
#    endif ()
#endfunction()
#The deploy function uses a Python script to resolve the transitive dependencies.
function(deploy target destination)
    if (WIN32)  # MSVC or MINGW
        add_custom_command(TARGET ${target} POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E make_directory ${destination}
                COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${target}> ${destination}
                COMMAND ${CMAKE_COMMAND} -E echo "Copying runtime DLLs of ${target} to ${destination}..."
                COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_LIST_DIR}/deploy.py $<TARGET_FILE:${target}>  ${destination}
                #ToDo: change PYTHON_RUNTIME_DLL to PYTHON_LIBRARIES, so that it works on all platforms?
                COMMAND ${CMAKE_COMMAND} -E echo "Copying Python runtime DLL ${PYTHON_RUNTIME_DLL} to ${destination}..."
                COMMAND ${CMAKE_COMMAND} -E copy_if_different ${PYTHON_RUNTIME_DLL} ${destination}
                COMMAND_EXPAND_LISTS
        )
    else () # macOS and Linux
        add_custom_command(TARGET ${target} POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E make_directory ${destination}
                COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${target}> ${destination}
                COMMAND ${CMAKE_COMMAND} -E echo "Copying runtime dependencies of ${target} to ${destination}..."
                COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_LIST_DIR}/deploy.py $<TARGET_FILE:${target}>  ${destination}
                COMMAND ${CMAKE_COMMAND} -E echo "Copying Python runtime library ${PYTHON_LIBRARIES} to ${destination}..."
                COMMAND ${CMAKE_COMMAND} -E copy_if_different ${PYTHON_LIBRARIES} ${destination}
                COMMAND_EXPAND_LISTS
        )
    endif ()
endfunction()
# Use the above function to deploy polyfit python module and its dependencies to the Python package directory.
set(POLYFIT_PYTHON_PACKAGE_DIR "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/python")
deploy(${PROJECT_NAME} ${POLYFIT_PYTHON_PACKAGE_DIR}/polyfit)

# Configure and create pyproject.toml
configure_file(${CMAKE_CURRENT_LIST_DIR}/pyproject.toml.in ${POLYFIT_PYTHON_PACKAGE_DIR}/pyproject.toml @ONLY)

# Copy the readme file
file(COPY ${CMAKE_CURRENT_LIST_DIR}/README.md DESTINATION ${POLYFIT_PYTHON_PACKAGE_DIR})
# Copy the license file
file(COPY ${POLYFIT_ROOT_DIR}/LICENSE DESTINATION ${POLYFIT_PYTHON_PACKAGE_DIR}/polyfit)

# Generate the __init__.py file for the Python package
file(WRITE ${POLYFIT_PYTHON_PACKAGE_DIR}/polyfit/__init__.py
        "# Alias for the ${POLYFIT_MODULE_NAME} module\n"
        "import os\n"
        "\n"
        "try:\n"
        "    from .${POLYFIT_MODULE_NAME} import *\n"
        "except ImportError:\n"
        "    from ${POLYFIT_MODULE_NAME} import *\n"
        )