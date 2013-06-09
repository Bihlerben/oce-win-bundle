CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

####### FREEIMAGE
PROJECT(tbb)

SET(CMAKE_DEBUG_POSTFIX "_debug")

MACRO(ASSEMBLE_FILES)
IF(MSVC) #fix for broken CMake ASM support
	SET(OBJS)
	FOREACH(SOURCE ${ARGN})
			MESSAGE(${SOURCE})
			GET_FILENAME_COMPONENT(SOURCE ${SOURCE} ABSOLUTE)
			GET_FILENAME_COMPONENT(DEST ${SOURCE} NAME_WE)
			SET(DEST ${CMAKE_CURRENT_BINARY_DIR}/${DEST}.obj)
			ADD_CUSTOM_COMMAND (OUTPUT ${DEST}
					COMMAND echo "Assembling ${SOURCE} to ${DEST}"
					COMMAND ${CMAKE_ASM_MASM_COMPILER} /safeseh /c ${SOURCE} ${DEST} 
					DEPENDS ${SOURCE}
					MAIN_DEPENDENCY ${SOURCE}
					)
			SET(OBJS ${OBJS} ${DEST})
	ENDFOREACH()
ENDIF()
ENDMACRO()

IF(MSVC)
	ADD_DEFINITIONS("-D_CRT_SECURE_NO_DEPRECATE")
	ADD_DEFINITIONS("-D_CRT_NONSTDC_NO_WARNINGS")
	ADD_DEFINITIONS("-D_WINDLL")
	ENABLE_LANGUAGE(ASM_MASM)
ENDIF()

IF(WIN32)
	ADD_DEFINITIONS("-D__TBB_BUILD" "-DDO_ITT_NOTIFY" "-DUSE_WINTHREAD")
	SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -DTBB_USE_DEBUG -D__TBB_NO_IMPLICIT_LINKAGE")
	SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -D__TBB_NO_IMPLICIT_LINKAGE")
ENDIF()

INCLUDE_DIRECTORIES(${TBB_DIR}/src)
INCLUDE_DIRECTORIES(${TBB_DIR}/include)
INCLUDE_DIRECTORIES(${TBB_DIR}/src/rml/include)
INCLUDE_DIRECTORIES(${TBB_DIR}/src/tbb)

IF(MSVC)
	INCLUDE_DIRECTORIES(${TBB_DIR}/build/vsproject)
ENDIF()

FILE(GLOB src_files ${TBB_DIR}/src/tbb/*.cpp)
FILE(GLOB old_files ${TBB_DIR}/src/old/*.cpp)
SET(src_rml_files ${TBB_DIR}/src/rml/client/rml_tbb.cpp )

LIST(REMOVE_ITEM old_files "${TBB_DIR}/src/old/test_concurrent_queue_v2.cpp")
LIST(REMOVE_ITEM old_files "${TBB_DIR}/src/old/test_concurrent_vector_v2.cpp")
LIST(REMOVE_ITEM old_files "${TBB_DIR}/src/old/test_mutex_v2.cpp")
LIST(REMOVE_ITEM old_files "${TBB_DIR}/src/old/test_task_scheduler_observer_v3.cpp")

IF(WIN32 AND MSVC)
   IF(${BIT} EQUAL "32")
	FILE(GLOB platform_files ${TBB_DIR}/src/tbb/ia32-masm/*.asm)
   ELSE()
    FILE(GLOB platform_files ${TBB_DIR}/src/tbb/intel64-masm/*.asm)
   ENDIF()   
ENDIF()

ASSEMBLE_FILES(${platform_files})

IF(MSVC)
	SET(def_file ${CMAKE_CURRENT_BINARY_DIR}/tbb.def)
	IF(${BIT} EQUAL "32")
		SET(src_def ${TBB_DIR}/src/tbb/win32-tbb-export.def)
	ELSE()
		SET(src_def ${TBB_DIR}/src/tbb/win64-tbb-export.def)
	ENDIF()

	ADD_CUSTOM_COMMAND(COMMAND echo "Generating DEF file"
					COMMAND ${CMAKE_CXX_COMPILER} /nologo /TC /EP ${src_def} /I${TBB_DIR}/include /DTBB_USE_DEBUG /DDO_ITT_NOTIFY /DUSE_WINTHREAD /D_CRT_SECURE_NO_DEPRECATE /D_WIN32_WINNT=0x0400 >"${def_file}"
					DEPENDS ${src_def}
					OUTPUT ${def_file}
				)
ENDIF()


ADD_LIBRARY (tbb ${OCEBUNDLE_ADD_LIB_PREFIX} ${src_files} ${src_rml_files} ${OBJS} ${def_file} ${old_files})

# INSTALL
IF(OCEBUNDLE_AUTO_INSTALL)

	FILE(GLOB tbb_headers ${TBB_DIR}/include/*.h)
	
	INSTALL(FILES ${tbb_headers} DESTINATION ${OCEBUNDLE_INSTALL_DIR}/include/tbb)
	
	INSTALL(TARGETS tbb
			ARCHIVE DESTINATION  "${OCEBUNDLE_INSTALL_DIR_BITS}/lib/" CONFIGURATIONS Debug Release MinSizeRel RelWithDebInfo
			LIBRARY DESTINATION  "${OCEBUNDLE_INSTALL_DIR_BITS}/lib/" CONFIGURATIONS Debug Release MinSizeRel RelWithDebInfo
			RUNTIME DESTINATION  "${OCEBUNDLE_INSTALL_DIR_BITS}/bin/" CONFIGURATIONS Debug Release MinSizeRel RelWithDebInfo
			)
	
ENDIF()
