cmake_minimum_required(VERSION 3.5...3.22)

project(polyscope)

add_definitions(-DGLM_ENABLE_EXPERIMENTAL)

### Policy settings
cmake_policy(SET CMP0054 NEW)   # don't implicitly dereference inside if()

## Project options

# Backend
set(POLYSCOPE_BACKEND_OPENGL3_GLFW "ON" CACHE BOOL "Enable openGL3_glfw backend")
set(POLYSCOPE_BACKEND_OPENGL_MOCK "ON" CACHE BOOL "Enable openGL_mock backend")
set(POLYSCOPE_BACKEND_OPENGL3_EGL "AUTO" CACHE STRING "Enable openGL3_egl backend") # 'AUTO' means "if we're on linux and EGL.h is available"

### Do anything needed for dependencies and bring their stuff in to scope
add_subdirectory(deps)

### Add things for this project
add_subdirectory(src)
