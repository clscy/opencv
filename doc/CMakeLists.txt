#-----------------------
#  CMake file for OpenCV docs
#-----------------------

if(BUILD_DOCS AND DOXYGEN_FOUND)
  set(HAVE_DOC_GENERATOR TRUE)
else()
  set(HAVE_DOC_GENERATOR FALSE)
endif()

if(HAVE_DOC_GENERATOR)
  project(opencv_docs)
  # build lists of modules to be documented
  set(BASE_MODULES "")
  set(EXTRA_MODULES "")

  foreach(mod ${OPENCV_MODULES_BUILD} ${OPENCV_MODULES_DISABLED_USER} ${OPENCV_MODULES_DISABLED_AUTO} ${OPENCV_MODULES_DISABLED_FORCE})
    string(REGEX REPLACE "^opencv_" "" mod "${mod}")
    if("${OPENCV_MODULE_opencv_${mod}_LOCATION}" STREQUAL "${OpenCV_SOURCE_DIR}/modules/${mod}")
      list(APPEND BASE_MODULES ${mod})
    else()
      list(APPEND EXTRA_MODULES ${mod})
    endif()
  endforeach()
  ocv_list_sort(BASE_MODULES)
  ocv_list_sort(EXTRA_MODULES)
  set(FIXED_ORDER_MODULES core imgproc imgcodecs videoio highgui video calib3d features2d objdetect ml flann photo stitching)
  list(REMOVE_ITEM BASE_MODULES ${FIXED_ORDER_MODULES})
  set(BASE_MODULES ${FIXED_ORDER_MODULES} ${BASE_MODULES})
endif(HAVE_DOC_GENERATOR)

# ========= Doxygen docs =========

if(BUILD_DOCS AND DOXYGEN_FOUND)
  # not documented modules list
  list(APPEND blacklist "ts" "java" "python2" "python3" "world" "contrib_world")
  unset(CMAKE_DOXYGEN_TUTORIAL_CONTRIB_ROOT)

  # gathering headers
  set(paths_include)
  set(paths_doc)
  set(paths_bib)
  set(paths_sample)
  set(paths_tutorial)
  set(paths_hal_interface)
  set(refs_main)
  set(refs_extra)
  set(deps)
  foreach(m ${BASE_MODULES} ${EXTRA_MODULES})
    list(FIND blacklist ${m} _pos)
    if(${_pos} EQUAL -1)
      # include folder
      set(header_dir "${OPENCV_MODULE_opencv_${m}_LOCATION}/include")
      if(EXISTS "${header_dir}")
        list(APPEND paths_include "${header_dir}")
        list(APPEND deps ${header_dir})
      endif()
      # doc folder
      set(docs_dir "${OPENCV_MODULE_opencv_${m}_LOCATION}/doc")
      if(EXISTS "${docs_dir}")
        list(APPEND paths_doc "${docs_dir}")
        list(APPEND deps ${docs_dir})
      endif()
      # sample folder
      set(sample_dir "${OPENCV_MODULE_opencv_${m}_LOCATION}/samples")
      if(EXISTS "${sample_dir}")
        list(APPEND paths_sample "${sample_dir}")
        list(APPEND deps ${sample_dir})
      endif()
      # tutorial folder
      set(tutorial_dir "${OPENCV_MODULE_opencv_${m}_LOCATION}/tutorials")
      if(EXISTS "${tutorial_dir}")
        list(APPEND paths_tutorial "${tutorial_dir}")
        list(APPEND deps ${tutorial_dir})

        # tutorial reference entry
        file(GLOB tutorials RELATIVE "${OPENCV_MODULE_opencv_${m}_LOCATION}" "${tutorial_dir}/*.markdown")
        foreach (t ${tutorials})
          if (NOT DEFINED CMAKE_DOXYGEN_TUTORIAL_CONTRIB_ROOT)
            set(CMAKE_DOXYGEN_TUTORIAL_CONTRIB_ROOT "- @ref tutorial_contrib_root")
            set(tutorial_contrib_root "${CMAKE_CURRENT_BINARY_DIR}/contrib_tutorials.markdown")
            file(WRITE "${tutorial_contrib_root}"
              "Tutorials for contrib modules {#tutorial_contrib_root}\n"
              "=============================\n")
          endif()
          file(STRINGS "${OPENCV_MODULE_opencv_${m}_LOCATION}/${t}" tutorial_id LIMIT_COUNT 1 REGEX ".*{#[^}]+}")
          string(REGEX REPLACE ".*{#([^}]+)}" "\\1" tutorial_id "${tutorial_id}")
          file(APPEND "${tutorial_contrib_root}" "- ${m}. @subpage ${tutorial_id}\n")
        endforeach()
      endif()
      # HAL replacement file
      set(replacement_header "${OPENCV_MODULE_opencv_${m}_LOCATION}/src/hal_replacement.hpp")
      if(EXISTS "${replacement_header}")
        list(APPEND paths_hal_interface "${replacement_header}")
      endif()

      # BiBTeX file
      set(bib_file "${docs_dir}/${m}.bib")
      if(EXISTS "${bib_file}")
        set(paths_bib "${paths_bib} ${bib_file}")
        list(APPEND deps ${bib_file})
      endif()
      # Reference entry
    #   set(one_ref "@ref ${m} | ${m}\n")
      set(one_ref "\t- ${m}. @ref ${m}\n")
      list(FIND EXTRA_MODULES ${m} _pos)
      if(${_pos} EQUAL -1)
        set(refs_main "${refs_main}${one_ref}")
      else()
        set(refs_extra "${refs_extra}${one_ref}")
      endif()
    endif()
  endforeach()

  # fix references
  # set(ref_header "Module name | Folder\n----------- | ------")
  # if(refs_main)
  #    set(refs_main "### Main modules\n${ref_header}\n${refs_main}")
  # endif()
  # if(refs_extra)
  #   set(refs_extra "### Extra modules\n${ref_header}\n${refs_extra}")
  # endif()
  if(refs_main)
    set(refs_main "- Main modules:\n${refs_main}")
  endif()
  if(refs_extra)
    set(refs_extra "- Extra modules:\n${refs_extra}")
  endif()

  # additional config
  set(doxyfile "${CMAKE_CURRENT_BINARY_DIR}/Doxyfile")
  set(rootfile "${CMAKE_CURRENT_BINARY_DIR}/root.markdown")
  set(bibfile "${CMAKE_CURRENT_SOURCE_DIR}/opencv.bib")
  set(faqfile "${CMAKE_CURRENT_SOURCE_DIR}/faq.markdown")
  set(tutorial_path "${CMAKE_CURRENT_SOURCE_DIR}/tutorials")
  set(tutorial_py_path "${CMAKE_CURRENT_SOURCE_DIR}/py_tutorials")
  set(example_path "${CMAKE_SOURCE_DIR}/samples")

  # set export variables
  string(REPLACE ";" " \\\n" CMAKE_DOXYGEN_INPUT_LIST "${rootfile} ; ${faqfile} ; ${paths_include} ; ${paths_hal_interface} ; ${paths_doc} ; ${tutorial_path} ; ${tutorial_py_path} ; ${paths_tutorial} ; ${tutorial_contrib_root}")
  string(REPLACE ";" " \\\n" CMAKE_DOXYGEN_IMAGE_PATH "${paths_doc} ; ${tutorial_path} ; ${tutorial_py_path} ; ${paths_tutorial}")
  # TODO: remove paths_doc from EXAMPLE_PATH after face module tutorials/samples moved to separate folders
  string(REPLACE ";" " \\\n" CMAKE_DOXYGEN_EXAMPLE_PATH  "${example_path} ; ${paths_doc} ; ${paths_sample}")
  set(CMAKE_DOXYGEN_LAYOUT "${CMAKE_CURRENT_SOURCE_DIR}/DoxygenLayout.xml")
  set(CMAKE_DOXYGEN_OUTPUT_PATH "doxygen")
  set(CMAKE_DOXYGEN_MAIN_REFERENCE "${refs_main}")
  set(CMAKE_DOXYGEN_EXTRA_REFERENCE "${refs_extra}")
  set(CMAKE_EXTRA_BIB_FILES "${bibfile} ${paths_bib}")
  if (CMAKE_DOXYGEN_GENERATE_QHP)
    set(CMAKE_DOXYGEN_GENERATE_QHP "YES")
  else()
    set(CMAKE_DOXYGEN_GENERATE_QHP "NO")
  endif()

  list(APPEND CMAKE_DOXYGEN_HTML_FILES "${CMAKE_CURRENT_SOURCE_DIR}/opencv.ico")
  list(APPEND CMAKE_DOXYGEN_HTML_FILES "${CMAKE_CURRENT_SOURCE_DIR}/pattern.png")
  list(APPEND CMAKE_DOXYGEN_HTML_FILES "${CMAKE_CURRENT_SOURCE_DIR}/acircles_pattern.png")
  list(APPEND CMAKE_DOXYGEN_HTML_FILES "${CMAKE_CURRENT_SOURCE_DIR}/bodybg.png")
  list(APPEND CMAKE_DOXYGEN_HTML_FILES "${CMAKE_CURRENT_SOURCE_DIR}/mymath.sty")
  string(REPLACE ";" " \\\n" CMAKE_DOXYGEN_HTML_FILES "${CMAKE_DOXYGEN_HTML_FILES}")

  # writing file
  configure_file(Doxyfile.in ${doxyfile} @ONLY)
  configure_file(root.markdown.in ${rootfile} @ONLY)

  add_custom_target(doxygen
    COMMAND ${DOXYGEN_EXECUTABLE} ${doxyfile}
    DEPENDS ${doxyfile} ${rootfile} ${bibfile} ${deps}
  )
  install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doxygen/html
    DESTINATION "${OPENCV_DOC_INSTALL_PATH}"
    COMPONENT "docs" OPTIONAL
  )

  # Alias to build/install docs only
  add_custom_target(install_docs
      DEPENDS doxygen
      COMMAND "${CMAKE_COMMAND}" -DCMAKE_INSTALL_COMPONENT=docs -P "${CMAKE_BINARY_DIR}/cmake_install.cmake"
  )
endif()
