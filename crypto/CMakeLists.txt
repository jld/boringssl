include_directories(../include)

if(APPLE)
  if (${ARCH} STREQUAL "x86")
    set(PERLASM_FLAGS "-fPIC -DOPENSSL_IA32_SSE2")
  endif()
  set(PERLASM_STYLE macosx)
  set(ASM_EXT S)
  enable_language(ASM)
elseif(UNIX)
  if (${ARCH} STREQUAL "aarch64")
    # The "armx" Perl scripts look for "64" in the style argument
    # in order to decide whether to generate 32- or 64-bit asm.
    set(PERLASM_STYLE linux64)
  elseif (${ARCH} STREQUAL "arm")
    set(PERLASM_STYLE linux32)
  elseif (${ARCH} STREQUAL "x86")
    set(PERLASM_FLAGS "-fPIC -DOPENSSL_IA32_SSE2")
    set(PERLASM_STYLE elf)
  elseif (${ARCH} STREQUAL "ppc64le")
    set(PERLASM_STYLE ppc64le)
  else()
    set(PERLASM_STYLE elf)
  endif()
  set(ASM_EXT S)
  enable_language(ASM)
  set(CMAKE_ASM_FLAGS "${CMAKE_ASM_FLAGS} -Wa,--noexecstack")
else()
  if (CMAKE_CL_64)
    message("Using nasm")
    set(PERLASM_STYLE nasm)
  else()
    message("Using win32n")
    set(PERLASM_STYLE win32n)
    set(PERLASM_FLAGS "-DOPENSSL_IA32_SSE2")
  endif()

  # On Windows, we use the NASM output, specifically built with Yasm.
  set(ASM_EXT asm)
  enable_language(ASM_NASM)
endif()

function(perlasm dest src)
  add_custom_command(
    OUTPUT ${dest}
    COMMAND ${PERL_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/${src} ${PERLASM_STYLE} ${PERLASM_FLAGS} ${ARGN} ${dest}
    DEPENDS
    ${src}
    ${PROJECT_SOURCE_DIR}/crypto/perlasm/arm-xlate.pl
    ${PROJECT_SOURCE_DIR}/crypto/perlasm/ppc-xlate.pl
    ${PROJECT_SOURCE_DIR}/crypto/perlasm/x86_64-xlate.pl
    ${PROJECT_SOURCE_DIR}/crypto/perlasm/x86asm.pl
    ${PROJECT_SOURCE_DIR}/crypto/perlasm/x86gas.pl
    ${PROJECT_SOURCE_DIR}/crypto/perlasm/x86masm.pl
    ${PROJECT_SOURCE_DIR}/crypto/perlasm/x86nasm.pl
    WORKING_DIRECTORY .
  )
endfunction()

# Level 0.1 - depends on nothing outside this set.
add_subdirectory(stack)
add_subdirectory(lhash)
add_subdirectory(err)
add_subdirectory(buf)
add_subdirectory(base64)
add_subdirectory(bytestring)
add_subdirectory(pool)

# Level 0.2 - depends on nothing but itself
add_subdirectory(sha)
add_subdirectory(md4)
add_subdirectory(md5)
add_subdirectory(modes)
add_subdirectory(aes)
add_subdirectory(des)
add_subdirectory(rc4)
add_subdirectory(conf)
add_subdirectory(chacha)
add_subdirectory(poly1305)
add_subdirectory(curve25519)

# Level 1, depends only on 0.*
add_subdirectory(digest)
add_subdirectory(cipher)
add_subdirectory(rand)
add_subdirectory(bio)
add_subdirectory(bn)
add_subdirectory(obj)
add_subdirectory(asn1)

# Level 2
add_subdirectory(engine)
add_subdirectory(dh)
add_subdirectory(dsa)
add_subdirectory(rsa)
add_subdirectory(ec)
add_subdirectory(ecdh)
add_subdirectory(ecdsa)
add_subdirectory(hmac)

# Level 3
add_subdirectory(cmac)
add_subdirectory(evp)
add_subdirectory(hkdf)
add_subdirectory(pem)
add_subdirectory(x509)
add_subdirectory(x509v3)

# Level 4
add_subdirectory(pkcs8)

# Test support code
add_subdirectory(test)

add_library(
  crypto

  cpu-aarch64-linux.c
  cpu-arm.c
  cpu-arm-linux.c
  cpu-intel.c
  cpu-ppc64le.c
  crypto.c
  ex_data.c
  mem.c
  refcount_c11.c
  refcount_lock.c
  thread.c
  thread_none.c
  thread_pthread.c
  thread_win.c

  $<TARGET_OBJECTS:stack>
  $<TARGET_OBJECTS:lhash>
  $<TARGET_OBJECTS:err>
  $<TARGET_OBJECTS:base64>
  $<TARGET_OBJECTS:bytestring>
  $<TARGET_OBJECTS:pool>
  $<TARGET_OBJECTS:sha>
  $<TARGET_OBJECTS:md4>
  $<TARGET_OBJECTS:md5>
  $<TARGET_OBJECTS:digest>
  $<TARGET_OBJECTS:cipher>
  $<TARGET_OBJECTS:modes>
  $<TARGET_OBJECTS:aes>
  $<TARGET_OBJECTS:des>
  $<TARGET_OBJECTS:rc4>
  $<TARGET_OBJECTS:conf>
  $<TARGET_OBJECTS:chacha>
  $<TARGET_OBJECTS:poly1305>
  $<TARGET_OBJECTS:curve25519>
  $<TARGET_OBJECTS:buf>
  $<TARGET_OBJECTS:bn>
  $<TARGET_OBJECTS:bio>
  $<TARGET_OBJECTS:rand>
  $<TARGET_OBJECTS:obj>
  $<TARGET_OBJECTS:asn1>
  $<TARGET_OBJECTS:engine>
  $<TARGET_OBJECTS:dh>
  $<TARGET_OBJECTS:dsa>
  $<TARGET_OBJECTS:rsa>
  $<TARGET_OBJECTS:ec>
  $<TARGET_OBJECTS:ecdh>
  $<TARGET_OBJECTS:ecdsa>
  $<TARGET_OBJECTS:hmac>
  $<TARGET_OBJECTS:cmac>
  $<TARGET_OBJECTS:evp>
  $<TARGET_OBJECTS:hkdf>
  $<TARGET_OBJECTS:pem>
  $<TARGET_OBJECTS:x509>
  $<TARGET_OBJECTS:x509v3>
  $<TARGET_OBJECTS:pkcs8_lib>
)

if(NOT MSVC AND NOT ANDROID)
  target_link_libraries(crypto pthread)
endif()

add_executable(
  thread_test

  thread_test.c

  $<TARGET_OBJECTS:test_support>
)

target_link_libraries(thread_test crypto)
add_dependencies(all_tests thread_test)

add_executable(
  refcount_test

  refcount_test.cc
)

target_link_libraries(refcount_test crypto)
add_dependencies(all_tests refcount_test)

# TODO(davidben): Convert the remaining tests to GTest.
add_executable(
  crypto_test

  asn1/asn1_test.cc
  bio/bio_test.cc
  chacha/chacha_test.cc
  constant_time_test.cc
  curve25519/x25519_test.cc
  dh/dh_test.cc
  dsa/dsa_test.cc
  ec/ec_test.cc
  err/err_test.cc
  evp/evp_extra_test.cc
  rsa/rsa_test.cc

  $<TARGET_OBJECTS:gtest_main>
  $<TARGET_OBJECTS:test_support>
)

target_link_libraries(crypto_test crypto gtest)
if (WIN32)
  target_link_libraries(crypto_test ws2_32)
endif()
add_dependencies(all_tests crypto_test)
