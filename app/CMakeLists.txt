cmake_minimum_required(VERSION 3.4.1)

add_subdirectory(src/main/cpp/hl-pipelines)


add_library( HalideAndroidCamera2Gl
             SHARED
             src/main/cpp/AndroidBufferUtilities.cpp
			 src/main/cpp/HalideFilters.cpp
			 src/main/cpp/LockedSurface.cpp
			 src/main/cpp/YuvBufferT.cpp
			 )
			 
include_directories("${HALIDE_DIR}/include")

find_library( log-lib log )
find_library( m-lib m )
find_library( android-lib android )
find_library( EGL-lib EGL )
find_library( GLESv2-lib GLESv2 )



target_link_libraries( HalideAndroidCamera2Gl hl_pipelines ${m-lib} ${log-lib} ${android-lib} ${EGL-lib} ${GLESv2-lib} )