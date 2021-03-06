cmake_minimum_required(VERSION 3.5)
project(keystone C)
include(ProcessorCount)
set(CMAKE_VERBOSE_MAKEFILE ON)


###############################################################################
## MACROS
###############################################################################

# add patch macro
macro(add_patch submodule patch working_directory patch_list)
  add_custom_command(OUTPUT ${patch}.applied
    WORKING_DIRECTORY ${working_directory}
    COMMAND patch --forward -p0 < ${patchdir}/${submodule}/${patch} || true
    COMMAND touch ${CMAKE_BINARY_DIR}/${patch}.applied
    COMMENT "Applying ${patch}")
  list(APPEND ${patch_list} ${patch}.applied)
endmacro()

macro(mkdir name dir)
  add_custom_command(OUTPUT ${dir}/.exists
    COMMAND mkdir -p ${dir}
    COMMAND touch ${dir}/.exists
  )
  set(${name} ${dir})
  set(${name}_exists ${dir}/.exists)
endmacro()


###############################################################################
## CONFIGURATION/VARIABLES
###############################################################################

if(NOT DEFINED ENV{RISCV})
	message(FATAL_ERROR "set RISCV environment variable. \n Try `cd ${CMAKE_SOURCE_DIR}; source source.sh`")
endif()

set(USE_RUST_SM FALSE CACHE BOOL "Use Rust version of the security monitor.")
set(SM_CONFIGURE_ARGS --enable-opt=2 CACHE STRING "Security Monitor configure script arguments")
set(SM_PLATFORM "default" CACHE STRING "Board name for SM hardware-specific functions")
set(platform ${SM_PLATFORM})
message(STATUS "platform=${platform}")

set(LINUX_SIFIVE FALSE CACHE BOOL "Build linux for sifive")
set(sifive ${LINUX_SIFIVE})
message(STATUS "sifive=${sifive}")

if((CMAKE_BUILD_TYPE MATCHES "Debug") OR (CMAKE_BUILD_TYPE MATCHES "RelWithDebInfo"))
  message(STATUS "Using debug symbols")
  set(CFLAGS -g)
endif()

set(RISCV32 OFF CACHE BOOL "Build in 32-bit mode")
if(RISCV32)
  message(STATUS "riscv32")
  set(BITS 32)
  set(ABI ilp32d)
else()
  message(STATUS "riscv64")
  set(BITS 64)
  set(ABI lp64d)
endif()

set(ISA rv${BITS}imafdc)
set(confdir ${CMAKE_SOURCE_DIR}/conf)
set(patchdir ${CMAKE_SOURCE_DIR}/patches)
set(cross_compile riscv${BITS}-unknown-linux-gnu-)

set(bootrom_srcdir ${CMAKE_SOURCE_DIR}/bootrom)
set(bootrom_wrkdir ${CMAKE_BINARY_DIR}/bootrom.build)
set(qemu_wrkdir ${CMAKE_SOURCE_DIR}/qemu)
set(qemu_srcdir ${CMAKE_SOURCE_DIR}/qemu)
set(sm_srcdir ${CMAKE_SOURCE_DIR}/riscv-pk)
mkdir(sm_wrkdir ${CMAKE_BINARY_DIR}/riscv-pk.build)
set(buildroot_srcdir ${CMAKE_SOURCE_DIR}/buildroot)
set(buildroot_wrkdir ${CMAKE_BINARY_DIR}/buildroot.build)
set(buildroot_config ${confdir}/qemu_riscv${BITS}_virt_defconfig)
set(overlay_dir ${CMAKE_BINARY_DIR}/overlay)
set(overlay_root ${overlay_dir}/root)

if(firesim)
  message(STATUS "FireSim configs and patches. Forcing initramfs=y")
	set(linux_defconfig ${confdir}/linux64-firesim-defconfig)
  message(STATUS "Using Linux defconfig: ${linux_defconfig}")
	set(initramfs true)
elseif(sifive)
  message(STATUS "SiFive Unleashed configs and patches. Forcing initramfs=y")
	set(linux_defconfig ${confdir}/linux64-sifive-defconfig)
  message(STATUS "Using Linux defconfig: ${linux_defconfig}")
	set(initramfs true)
endif()

if(NOT DEFINED linux_defconfig)
	set(linux_defconfig ${confdir}/linux${BITS}-defconfig)
endif()

set(linux_srcdir ${CMAKE_SOURCE_DIR}/linux)
mkdir(linux_wrkdir ${CMAKE_BINARY_DIR}/linux.build)
set(linux_vmlinux ${linux_wrkdir}/vmlinux)
set(linux_vmlinux_stripped ${linux_wrkdir}/vmlinux-stripped)
set(driver_srcdir ${CMAKE_SOURCE_DIR}/linux-keystone-driver)
set(driver_wrkdir ${CMAKE_BINARY_DIR}/linux-keystone-driver.build)
set(tests_srcdir ${CMAKE_SOURCE_DIR}/tests)
set(tests_wrkdir ${CMAKE_BINARY_DIR}/tests.build)
set(final_image ${CMAKE_BINARY_DIR}/bbl.bin)
set(initramfs_sysroot ${CMAKE_BINARY_DIR}/initramfs-sysroot)


###############################################################################
## COMPONENT: QEMU
###############################################################################

set(qemu_system ${qemu_wrkdir}/riscv${BITS}-softmmu/qemu-system-riscv${BITS})
add_patch("qemu" "qemu-pmp-bug.patch" ${qemu_srcdir} qemu_patches)
add_patch("qemu" "qemu-secure-boot.patch" ${qemu_srcdir} qemu_patches)

add_custom_target("qemu" ALL DEPENDS ${qemu_system})
add_custom_command(OUTPUT ${qemu_system}
  COMMAND $(MAKE) -C ${qemu_srcdir}
  DEPENDS ${qemu_wrkdir}/config-host.mak ${qemu_patches}
  COMMENT "Building QEMU"
)
add_custom_command(OUTPUT ${qemu_wrkdir}/config-host.mak
  WORKING_DIRECTORY ${qemu_srcdir}
  COMMAND ./configure --target-list=riscv${BITS}-softmmu,riscv${BITS}-linux-user
  COMMENT "Configuring QEMU"
)


###############################################################################
## COMPONENT: buildroot
###############################################################################

add_custom_command(OUTPUT ${overlay_root} COMMAND mkdir -p ${overlay_root})
add_custom_command(OUTPUT ${overlay_dir} COMMAND mkdir -p ${overlay_dir})
add_custom_command(OUTPUT ${buildroot_wrkdir} COMMAND mkdir -p ${buildroot_wrkdir})
add_custom_target("buildroot" ALL DEPENDS ${buildroot_srcdir} ${buildroot_wrkdir}/.config ${overlay_root} ${buildroot_wrkdir}
  COMMAND $(MAKE) -s -C ${buildroot_srcdir} RISCV=$ENV{RISCV} PATH=$ENV{PATH} O=${buildroot_wrkdir}
  COMMENT "Building buildroot"
)

string(REPLACE "/" "\\/" overlay_dir_stripped ${overlay_dir})
add_custom_command(DEPENDS ${buildroot_config} OUTPUT ${buildroot_wrkdir}/.config
  COMMAND mkdir -p ${buildroot_wrkdir}
  COMMAND cp ${buildroot_config} ${buildroot_wrkdir}/.config
  COMMAND sed \"s/^BR2_ROOTFS_OVERLAY=.*/BR2_ROOTFS_OVERLAY=\\\"${overlay_dir_stripped}\\\"/g\" -i ${buildroot_wrkdir}/.config
  COMMAND $(MAKE) -s -C ${buildroot_srcdir} RISCV=$ENV{RISCV} PATH=$ENV{PATH}
    O=${buildroot_wrkdir} CROSS_COMPILE=${cross_compile} olddefconfig
  COMMENT "Configuring buildroot (overlay = ${overlay_dir})"
)


###############################################################################
## component: bootrom
###############################################################################

add_custom_command(OUTPUT ${bootrom_wrkdir} COMMAND mkdir -p ${bootrom_wrkdir})
add_custom_target("bootrom" ALL
  COMMAND $(MAKE) -C ${bootrom_srcdir} O=${bootrom_wrkdir}
  DEPENDS ${bootrom_wrkdir} ${bootrom_srcdir}
  COMMENT "Building bootrom"
)


###############################################################################
## COMPONENT: linux kernel 
###############################################################################

add_custom_command(OUTPUT ${linux_wrkdir}/.config DEPENDS ${linux_defconfig} ${linux_wrkdir_exists}
  COMMAND cp ${linux_defconfig} ${linux_wrkdir}/.config
  COMMAND $(MAKE) -C ${linux_srcdir} O=${linux_wrkdir} ARCH=riscv olddefconfig
  COMMENT "Configuring linux"
)

if(firesim)
  add_patch("linux" "linux${BITS}.firesim.patch" ${linux_srcdir} linux_patches)
elseif(sifive)
  add_patch("linux" "linux${BITS}.sifive.patch" ${linux_srcdir} linux_patches)
else()
  add_patch("linux" "linux${BITS}.patch" ${linux_srcdir} linux_patches)
endif()


if(initramfs)
  # linux-initramfs
  execute_process(COMMAND id -u OUTPUT_VARIABLE uid)
  string(STRIP ${uid} uid)
  execute_process(COMMAND id -g OUTPUT_VARIABLE gid)
  string(STRIP ${gid} gid)
  add_custom_command(OUTPUT ${initramfs_sysroot} COMMAND mkdir -p ${initramfs_sysroot})
  add_custom_target("sysroot" DEPENDS "buildroot" ${initramfs_sysroot} ${overlay_dir}
    COMMAND tar -xpf ${buildroot_wrkdir}/images/rootfs.tar -C ${initramfs_sysroot} --exclude ./dev --exclude ./usr/share/locale
    COMMAND echo "::sysinit:/bin/mount -t devtmpfs devtmpfs /dev" >> ${initramfs_sysroot}/etc/inittab
    )
  add_custom_command(OUTPUT ${linux_vmlinux_stripped} ${linux_vmlinux} DEPENDS "sysroot" ${linux_srcdir} ${linux_wrkdir}/.config ${linux_patches}
    COMMAND $(MAKE) -C ${linux_srcdir}
      O=${linux_wrkdir} CONFIG_INITRAMFS_SOURCE="${confdir}/initramfs.txt ${initramfs_sysroot}"
      CONFIG_INITRAMFS_ROOT_UID=${uid} CONFIG_INITRAMFS_ROOT_GID=${gid}
      CONFIG_DEVTMPFS=y CONFIG_DEVTMPFS_MOUNT=y
      CROSS_COMPILE=${cross_compile} ARCH=riscv vmlinux
    COMMAND ${cross_compile}strip -o ${linux_vmlinux_stripped} ${linux_vmlinux}
    COMMENT "Building linux (initramfs)"
  )
else()
  add_custom_command(OUTPUT ${linux_vmlinux_stripped} ${linux_vmlinux} DEPENDS ${linux_srcdir} ${linux_wrkdir}/.config ${linux_patches}
    COMMAND $(MAKE) -C ${linux_srcdir} O=${linux_wrkdir} CROSS_COMPILE=${cross_compile} ARCH=riscv vmlinux
    COMMAND ${cross_compile}strip -o ${linux_vmlinux_stripped} ${linux_vmlinux}
    COMMENT "Building linux"
  )
endif()

add_custom_target("linux" ALL DEPENDS ${linux_vmlinux_stripped} ${linux_vmlinux})

###############################################################################
## COMPONENT: linux driver
###############################################################################

add_custom_command(OUTPUT ${driver_wrkdir} COMMAND mkdir -p ${driver_wrkdir})
add_custom_target("driver-sync" DEPENDS ${driver_srcdir} ${driver_wrkdir}
	COMMAND rsync -r ${driver_srcdir}/ ${driver_wrkdir})
add_custom_target("driver" ALL DEPENDS ${driver_srcdir} ${linux_srcdir} "linux" "driver-sync"
  COMMAND $(MAKE) -C ${linux_wrkdir} O=${linux_wrkdir} CROSS_COMPILE=${cross_compile} ARCH=riscv
    M=${driver_wrkdir} modules
  COMMENT "Building driver"
)


###############################################################################
## COMPONENT: bbl + sm
###############################################################################

if (${USE_RUST_SM})
  set(enabled_sm --enable-sm_rs)
else()
  set(enabled_sm --enable-sm)
endif()

add_custom_command(OUTPUT ${sm_wrkdir}/Makefile WORKING_DIRECTORY ${sm_wrkdir}
  DEPENDS ${sm_wrkdir_exists} "linux"
  COMMAND ${sm_srcdir}/configure --host=riscv${BITS}-unknown-linux-gnu --with-payload=${linux_vmlinux_stripped}
    --enable-logo --with-logo=${confdir}/sifive_logo.txt ${enabled_sm} ${SM_CONFIGURE_ARGS} --with-target-platform=${platform}
  COMMENT "Configuring sm"
)
add_custom_target("sm" ALL DEPENDS ${sm_wrkdir}/Makefile WORKING_DIRECTORY ${sm_wrkdir}
  COMMAND env CFLAGS='${CFLAGS} -mabi=${ABI} -march=${ISA}' $(MAKE) -C ${sm_wrkdir}
  COMMENT "Building sm"
)

if(firesim)
  add_patch("riscv-pk" "riscv-pk.firesim.patch" ${sm_srcdir} "sm")
elseif(sifive)
  add_patch("riscv-pk" "riscv-pk.sifive.patch" ${sm_srcdir} "sm")
endif()


###############################################################################
## COMPONENT: tests
###############################################################################

add_custom_command(OUTPUT ${tests_wrkdir} COMMAND mkdir -p ${tests_wrkdir})
add_custom_target("tests" DEPENDS ${tests_srcdir} ${tests_wrkdir}
  COMMAND OUTPUT_DIR=${overlay_root} ARTIFACTS_DIR=${tests_wrkdir} ${tests_srcdir}/tests/vault.sh
  COMMENT "Building tests"
)


###############################################################################
## COMPONENT: image
###############################################################################

add_custom_target("image-deps" DEPENDS "sm" "tests" "driver"
  COMMAND find ${driver_wrkdir} -name "*.ko" -exec cp {} ${overlay_root} \\\\;
)
add_custom_target("image" DEPENDS "buildroot"
  COMMAND ${cross_compile}objcopy -S -O binary --change-addresses -0x80000000 ${sm_wrkdir}/bbl ${final_image}
  COMMENT "Generating image"
)
add_dependencies("buildroot" "image-deps")


###############################################################################
## QEMU scripts
###############################################################################

# scripts
set(scripts ${CMAKE_BINARY_DIR}/scripts)
add_custom_command(OUTPUT ${scripts} COMMAND mkdir -p ${scripts})

# if initramfs is false, we need to tell qemu where to find the block device
if(initramfs)
  set(extra_qemu_options "")
else()
  set(extra_qemu_options "\
      -append \"console=ttyS0 ro root=/dev/vda\" \
      -drive file=${buildroot_wrkdir}/images/rootfs.ext2,format=raw,id=hd0 \
      -device virtio-blk-device,drive=hd0 \
  ")
endif()

# generate the qemu runscript, using the above options
add_custom_command(OUTPUT ${scripts}/run-qemu.sh DEPENDS ${scripts}
    WORKING_DIRECTORY ${scripts}
    COMMAND echo "\
      export HOST_PORT=\${HOST_PORT:=\"\$((3000 + RANDOM % 3000))\"}; \
      echo \"**** Running QEMU SSH on port \${HOST_PORT} ****\"; \
      export SMP=1; \
      while [ \"\$1\" != \"\" ]; do if [ \"\$1\" = \"-debug\" ]; then DEBUG=\"-gdb tcp::\$((HOST_PORT + 1)) -S -d in_asm -D debug.log\"; fi; if [ \"\$1\" = \"-smp\" ]; then SMP=\"\$2\"; shift; fi; shift; done; \
      ${qemu_system} \
      \$DEBUG \
      -m 2G \
      -nographic \
      -machine virt \
      -bios ${bootrom_wrkdir}/bootrom.bin \
      -kernel ${sm_wrkdir}/bbl \
      ${extra_qemu_options} \
      -netdev user,id=net0,net=192.168.100.1/24,dhcpstart=192.168.100.128,hostfwd=tcp::\$\{HOST_PORT\}-:22 \
      -device virtio-net-device,netdev=net0 \
      -device virtio-rng-pci \
      -smp $SMP" > run-qemu.sh
      VERBATIM
    COMMAND
      chmod +x run-qemu.sh
  )
add_custom_command(OUTPUT ${scripts}/test-qemu.sh DEPENDS ${CMAKE_SOURCE_DIR}/scripts ${scripts}
  COMMAND cp ${CMAKE_SOURCE_DIR}/scripts/test-qemu.sh ${scripts})
add_custom_command(OUTPUT ${scripts}/travis.sh DEPENDS ${CMAKE_SOURCE_DIR}/scripts ${scripts}
  COMMAND cp ${CMAKE_SOURCE_DIR}/scripts/travis.sh ${scripts})
add_custom_command(OUTPUT ${scripts}/gdb.sh DEPENDS ${CMAKE_SOURCE_DIR}/scripts ${scripts}
  COMMAND cp ${CMAKE_SOURCE_DIR}/scripts/gdb.sh ${scripts})

add_custom_target(
  "tools" ALL
  DEPENDS ${scripts} ${scripts}/run-qemu.sh ${scripts}/test-qemu.sh ${scripts}/travis.sh ${scripts}/gdb.sh
  COMMENT "Generating scripts and tools"
)

add_custom_target(
  "run-tests"
  DEPENDS "tools" "image"
  COMMAND
    ./scripts/travis.sh ${CMAKE_SOURCE_DIR}/tests
  COMMENT "Running tests"
)
