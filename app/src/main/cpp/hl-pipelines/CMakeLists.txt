cmake_minimum_required(VERSION 3.1)

if(NOT HALIDE_DIR)
	if($ENV{HALIDE_DIR})
			set(HALIDE_DIR, $ENV{HALIDE_DIR})
	else()
			message(FATAL_ERROR "HALIDE_DIR not set" )
	endif()
endif()

if(NOT CMAKE_HOST_WIN32)
	set(hlPipelineCompiler ${CMAKE_CXX_COMPILER})
	set(hlPipelineCompilerFlags "-std=c++11 -lpthread -ldl -lz")
	set(hlPipelineLibSuffix ".a")
	# shared import for generator run
	file(COPY $HALIDE_DIR/bin/libHalide.so
		DESTINATION ${CMAKE_CURRENT_LIST_DIR})
else()
	# clang's CLI is GCC-compatible
	if(NOT CLANG OR CLANG STREQUAL "")
		if(HOST_CLANG)
			set(CLANG "${HOST_CLANG}")
		elseif(ENV{CLANG})
			set(CLANG "$ENV{CLANG}")
		elseif(ENV{HOST_CLANG})
			set(CLANG "$ENV{HOST_CLANG}")
		else()
			message(FATAL_ERROR "CLANG not set" )
		endif()
	endif()
	
	set(hlPipelineCompiler ${CLANG})
	set(hlPipelineCompilerFlags "-std=c++14")

	
	# shared import for generator run
	file(COPY ${HALIDE_DIR}/bin/Release/Halide.dll
		DESTINATION ${CMAKE_CURRENT_LIST_DIR})
endif()


if(WIN32)
	set(hlPipelineLibSuffix ".lib")
else()
	set(hlPipelineLibSuffix ".a")
endif()

file(GLOB hlPipelineSources *.cpp)


# map CMAKE_SYSTEM_PROCESSOR to HL_TARGET `arch-bits`
if(${CMAKE_SYSTEM_PROCESSOR} STREQUAL "aarch64")
	set(hlArchBits "arm-64")
elseif(${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64")
	set(hlArchBits "x86-64")
elseif(${CMAKE_SYSTEM_PROCESSOR} STREQUAL AMD64)
	set(hlArchBits "x86-64")
else()
	message(FATAL_ERROR "unsupported arch '${CMAKE_SYSTEM_PROCESSOR}'")
endif()

if(ANDROID)
	set(hlTarget "${hlArchBits}-android")
else()
	set(hlTarget ${hlArchBits}) # using host OS
endif()


file(MAKE_DIRECTORY "gen")
file(WRITE gen/tool.cpp "#include \"Halide.h\"\nint main(int argc,char **argv){return Halide::Internal::generate_filter_main(argc, argv, std::cerr);}")
file(WRITE gen/dummy.c "int hl_pipelines_dummy_noop() {return 0;}")


get_filename_component(hlPipelineSourcesDir ${CMAKE_CURRENT_LIST_DIR} ABSOLUTE)


file(MAKE_DIRECTORY "${hlPipelineSourcesDir}/${hlTarget}")

set(hlPipelineLibs "")
set(hlPipelineHeaders "")


# makes sure headers are generated before building:
add_custom_target(hl_pipelines_headers ALL)

foreach(source IN LISTS hlPipelineSources)
    get_filename_component(obj ${source} NAME_WE)
    set(objLib ${hlPipelineSourcesDir}/${hlTarget}/${obj}${hlPipelineLibSuffix})
	set(objHeader ${hlPipelineSourcesDir}/${hlTarget}/${obj}.h)
    list(APPEND hlPipelineLibs ${objLib})
	list(APPEND hlPipelineHeaders ${objHeader})

    if(CMAKE_HOST_WIN32)
		set(objBin ${obj}_gen.exe)
		set(runCmd ${hlPipelineSourcesDir}/run_gen.cmd ${hlTarget} ${objBin})
	else()
		set(objBin ${obj}_gen)
		set(runCmd HL_TARGET=${hlTarget} ${objBin})
	endif()

	set(compileGenArgs ${source} gen/tool.cpp
		-I${HALIDE_DIR}/include -L${HALIDE_DIR}/lib -L${HALIDE_DIR}/lib/Release -lHalide ${hlPipelineCompilerFlags} -o ${objBin})
	
	
	# do 2 things (if pipeline source changed)
	# 1. compile generator to host-executable generator 
	# 2. run the generator
	add_custom_command(
		OUTPUT ${objLib} ${objHeader}
		COMMAND ${hlPipelineCompiler} ARGS ${compileGenArgs}
		COMMAND ${objBin} -g ${obj} -o ${hlTarget} target=${hlTarget}
		DEPENDS ${source}
		WORKING_DIRECTORY "${hlPipelineSourcesDir}"
		COMMENT "running ${obj} pipeline generator"
	)
	
	
	#if(NOT EXISTS ${obj}.h)
	#	add_custom_command(
	#		TARGET hl_pipelines_headers PRE_BUILD
	#		COMMAND ${hlPipelineCompiler} ARGS ${compileGenArgs}
	#		COMMAND ${objBin} -g ${obj} -o ${hlTarget} target=${hlTarget}
	#		COMMAND ${CMAKE_COMMAND} -E copy ${hlTarget}/${obj}.h ${obj}.h
	#		WORKING_DIRECTORY "${hlPipelineSourcesDir}"
	#		COMMENT "running ${obj} pipeline generator (header)"
	#	)
	#endif()
endforeach()




# we but the header here for dependencies
add_library(hl_pipelines STATIC gen/dummy.c ${hlPipelineHeaders})
target_link_libraries(hl_pipelines ${hlPipelineLibs})
set_target_properties(hl_pipelines PROPERTIES LINKER_LANGUAGE CXX)

add_dependencies(hl_pipelines hl_pipelines_headers)
