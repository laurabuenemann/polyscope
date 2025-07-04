# Maybe stop from CMAKEing in the wrong place
if (CMAKE_BINARY_DIR STREQUAL CMAKE_SOURCE_DIR)
    message(FATAL_ERROR "Source and build directories cannot be the same. Go use the /build directory.")
endif()

SET(INCLUDE_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../include/polyscope/")

# We need the shared openGL backend if either of these backends are enabled
if(POLYSCOPE_BACKEND_OPENGL3_GLFW OR POLYSCOPE_BACKEND_OPENGL3_EGL)
  SET(POLYSCOPE_BACKEND_OPENGL3 TRUE)
  add_definitions(-DPOLYSCOPE_BACKEND_OPENGL3_ENABLED)
else()
  SET(POLYSCOPE_BACKEND_OPENGL3 FALSE)
endif()

# Lists that will be populated for the render backend

# Add the main _engine file no matter what. All of the interesting parts are ifdef'd out, and this
# allows us to resolve stubs.
list (APPEND BACKEND_SRCS
    render/opengl/gl_engine.cpp
    render/opengl/gl_engine_glfw.cpp
    render/opengl/gl_engine_egl.cpp
    render/mock_opengl/mock_gl_engine.cpp
    render/opengl/shaders/texture_draw_shaders.cpp
    render/opengl/shaders/lighting_shaders.cpp
    render/opengl/shaders/grid_shaders.cpp
    render/opengl/shaders/ground_plane_shaders.cpp
    render/opengl/shaders/gizmo_shaders.cpp
    render/opengl/shaders/histogram_shaders.cpp
    render/opengl/shaders/surface_mesh_shaders.cpp
    render/opengl/shaders/volume_mesh_shaders.cpp
    render/opengl/shaders/vector_shaders.cpp
    render/opengl/shaders/sphere_shaders.cpp
    render/opengl/shaders/ribbon_shaders.cpp
    render/opengl/shaders/cylinder_shaders.cpp
    render/opengl/shaders/rules.cpp
    render/opengl/shaders/common.cpp
)

# Configure the render backend(s)
if("${POLYSCOPE_BACKEND_OPENGL3_GLFW}")
  message("Polyscope backend openGL3_glfw enabled")

  list(APPEND BACKEND_HEADERS
    ${INCLUDE_ROOT}render/opengl/gl_engine_glfw.h
    ${INCLUDE_ROOT}render/opengl/shaders/common.h
  )

  # Link settings
  list(APPEND BACKEND_LIBS
    glfw
  )

  if(APPLE)
      # On macOS, get openGL & friends from Frameworks; do not use GLAD at all

      # NOTE: This code is essentially duplicated here and in deps/imgui/CMakeLists.txt

      # Apple is playing hardball and deprecating openGL... we'll cross that bridge when we come to it
      # Silence warnings about openGL deprecation
      add_definitions(-DGL_SILENCE_DEPRECATION)

      find_library(cocoa_library Cocoa)
      find_library(opengl_library OpenGL)
      find_library(corevideo_library CoreVideo)
      find_library(iokit_library IOKit)
      list(APPEND BACKEND_LIBS ${cocoa_library} ${opengl_library} ${corevideo_library} ${iokit_library})
  else()
      # On Windows/Linux, use the glad openGL loader

      list(APPEND BACKEND_LIBS glad)
  endif()

  add_definitions(-DPOLYSCOPE_BACKEND_OPENGL3_GLFW_ENABLED)
endif()

## some logic to decide whether or not EGL is available and whether we want to build with it
include(CheckIncludeFileCXX)
CHECK_INCLUDE_FILE_CXX("EGL/egl.h" HAVE_EGL_LIB)
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  set(IS_LINUX TRUE)
else()
  set(IS_LINUX FALSE)
endif()

if(POLYSCOPE_BACKEND_OPENGL3_EGL STREQUAL "AUTO")
  if(IS_LINUX AND HAVE_EGL_LIB) # "AUTO" policy: only enable if we're on linux and EGL libs seem to be present
    set(POLYSCOPE_BACKEND_OPENGL3_EGL ON)
  else()
    set(POLYSCOPE_BACKEND_OPENGL3_EGL OFF)
  endif()
elseif(POLYSCOPE_BACKEND_OPENGL3_EGL) # if "TRUE" (or "ON" or "YES" etc etc)
  # sanity check that we can actually use the EGL backend
  if(NOT IS_LINUX)
    message(FATAL_ERROR "EGL backend is only supported on linux. Disable it with POLYSCOPE_BACKEND_OPENGL3_EGL=False")
  endif()
  if(NOT HAVE_EGL_LIB)
    message(FATAL_ERROR "EGL backend requires EGL headers, but 'EGL/egl.h' was not found. On Ubuntu, try 'apt-get install libegl1-mesa-dev'")
  endif()

  # we should be good-to-go for EGL

else()
  # nothing to do, already disabled
endif()




if("${POLYSCOPE_BACKEND_OPENGL3_EGL}")
  message("Polyscope backend openGL3_egl enabled")

  # this only supports headless rendering, and as of Mar 2024 headless rendering seems to only be available on nvidia drivers.

  list(APPEND BACKEND_HEADERS
    ${INCLUDE_ROOT}render/opengl/gl_engine_egl.h
    ${INCLUDE_ROOT}render/opengl/shaders/common.h
  )

  if(APPLE)
    message(FATAL_ERROR "Compiling EGL backed on APPLE is not supported. Set POLYSCOPE_BACKEND_OPENGL3_EGL=False")
  else()

    # On Windows/Linux, use the glad openGL loader
    list(APPEND BACKEND_LIBS glad)

    if(WIN32)
      message(FATAL_ERROR "Compiling EGL backed on Windows is not supported. Set POLYSCOPE_BACKEND_OPENGL3_EGL=False")
    else() # linux
      # only linux is actually supported for EGL

      # We intentionally *do not* list this as a shared library dependency, it gets loaded optionally at runtime
      # via dlopen(). See explanation in gl_engine_egl.cpp.
      # list(APPEND BACKEND_LIBS EGL) 
    endif()

  endif()

  add_definitions(-DPOLYSCOPE_BACKEND_OPENGL3_EGL_ENABLED)
endif()

if("${POLYSCOPE_BACKEND_OPENGL_MOCK}")
  message("Polyscope backend openGL_mock enabled")

  list(APPEND BACKEND_HEADERS
    ${INCLUDE_ROOT}render/mock_opengl/mock_gl_engine.h
    ${INCLUDE_ROOT}render/opengl/shaders/common.h
  )

  add_definitions(-DPOLYSCOPE_BACKEND_OPENGL_MOCK_ENABLED)
endif()


SET(SRCS

  # Core functionality
  polyscope.cpp
  options.cpp
  internal.cpp
  state.cpp
  structure.cpp
  quantity.cpp
  group.cpp
  utilities.cpp
  view.cpp
  screenshot.cpp
  messages.cpp
  pick.cpp
  widget.cpp

  # Rendering stuff
  render/engine.cpp
  render/color_maps.cpp
  render/ground_plane.cpp
  render/materials.cpp
  render/initialize_backend.cpp
  render/shader_builder.cpp
  render/managed_buffer.cpp
  render/templated_buffers.cpp

  # General utilities
  disjoint_sets.cpp
  file_helpers.cpp
  camera_parameters.cpp
  histogram.cpp
  persistent_value.cpp
  color_management.cpp
  transformation_gizmo.cpp
  slice_plane.cpp
  weak_handle.cpp
  marching_cubes.cpp
  elementary_geometry.cpp

  ## Structures

  # Point cloud
  point_cloud.cpp
  point_cloud_color_quantity.cpp
  point_cloud_scalar_quantity.cpp
  point_cloud_vector_quantity.cpp
  point_cloud_parameterization_quantity.cpp

  # Surface
  surface_mesh.cpp
  surface_color_quantity.cpp
  surface_scalar_quantity.cpp
  surface_vector_quantity.cpp
  surface_parameterization_quantity.cpp

  # Curve network
  curve_network.cpp
  curve_network_color_quantity.cpp
  curve_network_scalar_quantity.cpp
  curve_network_vector_quantity.cpp

  # Volume mesh
  volume_mesh.cpp
  volume_mesh_color_quantity.cpp
  volume_mesh_scalar_quantity.cpp
  volume_mesh_vector_quantity.cpp

  # Volume grid
  volume_grid.cpp
  volume_grid_scalar_quantity.cpp

  # Camera view
  camera_view.cpp

  # Simple triangle mesh
  simple_triangle_mesh.cpp

  # Floating quantities
  floating_quantity_structure.cpp
  floating_quantity.cpp
  image_quantity_base.cpp
  scalar_image_quantity.cpp
  color_image_quantity.cpp
  render_image_quantity_base.cpp
  depth_render_image_quantity.cpp
  color_render_image_quantity.cpp
  scalar_render_image_quantity.cpp
  raw_color_render_image_quantity.cpp
  raw_color_alpha_render_image_quantity.cpp

  # Rendering utilities
  imgui_config.cpp
  fullscreen_artist.cpp

  ## Embedded binary data
  render/bindata/bindata_font_lato_regular.cpp
  render/bindata/bindata_font_cousine_regular.cpp
  render/bindata/concrete_seamless.cpp
  render/bindata/bindata_clay.cpp
  render/bindata/bindata_wax.cpp
  render/bindata/bindata_candy.cpp
  render/bindata/bindata_flat.cpp
  render/bindata/bindata_mud.cpp
  render/bindata/bindata_ceramic.cpp
  render/bindata/bindata_jade.cpp
  render/bindata/bindata_normal.cpp
)


# Setting headers is useful for some build systems. For instance, in Visual Studio this is necessary for the header files to be indexed as a part of the project.
SET(HEADERS
  ${INCLUDE_ROOT}/affine_remapper.h
  ${INCLUDE_ROOT}/affine_remapper.ipp
  ${INCLUDE_ROOT}/camera_parameters.h
  ${INCLUDE_ROOT}/camera_parameters.ipp
  ${INCLUDE_ROOT}/camera_view.h
  ${INCLUDE_ROOT}/camera_view.ipp
  ${INCLUDE_ROOT}/check_invalid_values.h
  ${INCLUDE_ROOT}/color_management.h
  ${INCLUDE_ROOT}/color_image_quantity.h
  ${INCLUDE_ROOT}/color_render_image_quantity.h
  ${INCLUDE_ROOT}/colors.h
  ${INCLUDE_ROOT}/color_quantity.h
  ${INCLUDE_ROOT}/color_quantity.ipp
  ${INCLUDE_ROOT}/combining_hash_functions.h
  ${INCLUDE_ROOT}/context.h
  ${INCLUDE_ROOT}/curve_network.h
  ${INCLUDE_ROOT}/curve_network.ipp
  ${INCLUDE_ROOT}/curve_network_color_quantity.h
  ${INCLUDE_ROOT}/curve_network_quantity.h
  ${INCLUDE_ROOT}/curve_network_scalar_quantity.h
  ${INCLUDE_ROOT}/curve_network_vector_quantity.h
  ${INCLUDE_ROOT}/disjoint_sets.h
  ${INCLUDE_ROOT}/depth_render_image_quantity.h
  ${INCLUDE_ROOT}/elementary_geometry.h
  ${INCLUDE_ROOT}/file_helpers.h
  ${INCLUDE_ROOT}/floating_quantity_structure.h
  ${INCLUDE_ROOT}/floating_quantity.h
  ${INCLUDE_ROOT}/floating_quantities.h
  ${INCLUDE_ROOT}/group.h
  ${INCLUDE_ROOT}/histogram.h
  ${INCLUDE_ROOT}/image_quantity.h
  ${INCLUDE_ROOT}/imgui_config.h
  ${INCLUDE_ROOT}/implicit_helpers.h
  ${INCLUDE_ROOT}/implicit_helpers.ipp
  ${INCLUDE_ROOT}/messages.h
  ${INCLUDE_ROOT}/numeric_helpers.h
  ${INCLUDE_ROOT}/options.h
  ${INCLUDE_ROOT}/parameterization_quantity.h
  ${INCLUDE_ROOT}/parameterization_quantity.ipp
  ${INCLUDE_ROOT}/persistent_value.h
  ${INCLUDE_ROOT}/pick.h
  ${INCLUDE_ROOT}/pick.ipp
  ${INCLUDE_ROOT}/point_cloud.h
  ${INCLUDE_ROOT}/point_cloud.ipp
  ${INCLUDE_ROOT}/point_cloud_color_quantity.h
  ${INCLUDE_ROOT}/point_cloud_quantity.h
  ${INCLUDE_ROOT}/point_cloud_scalar_quantity.h
  ${INCLUDE_ROOT}/point_cloud_parameterization_quantity.h
  ${INCLUDE_ROOT}/point_cloud_vector_quantity.h
  ${INCLUDE_ROOT}/polyscope.h
  ${INCLUDE_ROOT}/quantity.h
  ${INCLUDE_ROOT}/quantity.ipp
  ${INCLUDE_ROOT}/raw_color_render_image_quantity.h
  ${INCLUDE_ROOT}/render/color_maps.h
  ${INCLUDE_ROOT}/render/engine.h
  ${INCLUDE_ROOT}/render/engine.ipp
  ${INCLUDE_ROOT}/render/ground_plane.h
  ${INCLUDE_ROOT}/render/managed_buffer.h
  ${INCLUDE_ROOT}/render/managed_buffer.ipp
  ${INCLUDE_ROOT}/render/material_defs.h
  ${INCLUDE_ROOT}/render/materials.h
  ${INCLUDE_ROOT}/render_image_quantity_base.h
  ${INCLUDE_ROOT}/scaled_value.h
  ${INCLUDE_ROOT}/scalar_quantity.h
  ${INCLUDE_ROOT}/scalar_quantity.ipp
  ${INCLUDE_ROOT}/screenshot.h
  ${INCLUDE_ROOT}/simple_triangle_mesh.h
  ${INCLUDE_ROOT}/simple_triangle_mesh.ipp
  ${INCLUDE_ROOT}/slice_plane.h
  ${INCLUDE_ROOT}/standardize_data_array.h
  ${INCLUDE_ROOT}/structure.h
  ${INCLUDE_ROOT}/structure.ipp
  ${INCLUDE_ROOT}/surface_color_quantity.h
  ${INCLUDE_ROOT}/surface_mesh.h
  ${INCLUDE_ROOT}/surface_mesh.ipp
  ${INCLUDE_ROOT}/surface_mesh_quantity.h
  ${INCLUDE_ROOT}/surface_parameterization_quantity.h
  ${INCLUDE_ROOT}/surface_scalar_quantity.h
  ${INCLUDE_ROOT}/surface_vector_quantity.h
  ${INCLUDE_ROOT}/texture_map_quantity.h
  ${INCLUDE_ROOT}/texture_map_quantity.ipp
  ${INCLUDE_ROOT}/types.h
  ${INCLUDE_ROOT}/utilities.h
  ${INCLUDE_ROOT}/view.h
  ${INCLUDE_ROOT}/vector_quantity.h
  ${INCLUDE_ROOT}/vector_quantity.ipp
  ${INCLUDE_ROOT}/volume_mesh.h
  ${INCLUDE_ROOT}/volume_mesh.ipp
  ${INCLUDE_ROOT}/volume_mesh_quantity.h
  ${INCLUDE_ROOT}/volume_mesh_scalar_quantity.h
  ${INCLUDE_ROOT}/volume_mesh_color_quantity.h
  ${INCLUDE_ROOT}/volume_mesh_vector_quantity.h
  ${INCLUDE_ROOT}/volume_grid.h
  ${INCLUDE_ROOT}/volume_grid.ipp
  ${INCLUDE_ROOT}/volume_grid_quantity.h
  ${INCLUDE_ROOT}/volume_grid_scalar_quantity.h
  ${INCLUDE_ROOT}/weak_handle.h
)

# Create a single library for the project
add_library(polyscope ${SRCS} ${BACKEND_SRCS} ${HEADERS} ${BACKEND_HEADERS})

# Mark the library target and its headers as installable
install(TARGETS polyscope)
install(DIRECTORY ${INCLUDE_ROOT}/ DESTINATION "${CMAKE_INSTALL_PREFIX}/include/polyscope")

# Required compiler settings
set_property(TARGET polyscope PROPERTY CXX_STANDARD 11)
set_property(TARGET polyscope PROPERTY CXX_STANDARD_REQUIRED TRUE)
set_target_properties(polyscope PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
add_definitions(-DNOMINMAX)

# Include settings
target_include_directories(polyscope PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/../include")

# Link settings
find_package(glm REQUIRED)
target_link_libraries(polyscope PUBLIC imgui glm::glm)
target_link_libraries(polyscope PRIVATE "${BACKEND_LIBS}" stb nlohmann_json::nlohmann_json MarchingCube::MarchingCube)
