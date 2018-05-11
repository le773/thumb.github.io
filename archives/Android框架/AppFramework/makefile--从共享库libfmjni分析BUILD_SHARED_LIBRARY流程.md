
## 1.1 libfmjni 编译流程
源码路径`packages/apps/FMRadio/jni/fmr/Android.mk`

```
Android.mk
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_SRC_FILES := \ // 依赖的源文件
    fmr_core.cpp \
    fmr_err.cpp \
    libfm_jni.cpp \
    common.cpp

LOCAL_C_INCLUDES := $(JNI_H_INCLUDE) \ // C头文件路径
    frameworks/base/include/media

LOCAL_SHARED_LIBRARIES := \ // 依赖的共享库
    libcutils \
    libdl \
    libmedia \

LOCAL_MODULE := libfmjni // 模块名
include $(BUILD_SHARED_LIBRARY)
```

## 1.2 BUILD_SHARED_LIBRARY
`BUILD_SHARED_LIBRARY`定义的路径`build/core/config.mk`

```
BUILD_SHARED_LIBRARY:= $(BUILD_SYSTEM)/shared_library.mk
```
## 1.3  shared_library.mk include 依赖关系

```
include $(BUILD_SYSTEM)/multilib.mk
include $(BUILD_SYSTEM)/module_arch_supported.mk
include $(BUILD_SYSTEM)/shared_library_internal.mk
include $(BUILD_SYSTEM)/module_arch_supported.mk
include $(BUILD_SYSTEM)/shared_library_internal.mk
include $(BUILD_COPY_HEADERS)
```

## 1.4  import_includes


###### 日志
```
Import includes file: obj/SHARED_LIBRARIES/libfmjni_intermediates/import_includes
core/binary.mk
```
###### 源码分析
```
import_includes := $(intermediates)/import_includes

####################################################
## Import includes
####################################################
import_includes := $(intermediates)/import_includes
import_includes_deps := $(strip \
    $(foreach l, $(installed_shared_library_module_names), \
      $(call intermediates-dir-for,SHARED_LIBRARIES,$(l),$(LOCAL_IS_HOST_MODULE),,$(LOCAL_2ND_ARCH_VAR_PREFIX))/export_includes) \
    $(foreach l, $(my_static_libraries) $(my_whole_static_libraries), \
      $(call intermediates-dir-for,STATIC_LIBRARIES,$(l),$(LOCAL_IS_HOST_MODULE),,$(LOCAL_2ND_ARCH_VAR_PREFIX))/export_includes))
$(import_includes): PRIVATE_IMPORT_EXPORT_INCLUDES := $(import_includes_deps)
$(import_includes) : $(LOCAL_MODULE_MAKEFILE) $(import_includes_deps)
        @echo Import includes file: $@
        $(hide) mkdir -p $(dir $@) && rm -f $@
ifdef import_includes_deps
        $(hide) for f in $(PRIVATE_IMPORT_EXPORT_INCLUDES); do \
          cat $$f >> $@; \
        done
else
        $(hide) touch $@
endif
```

## 1.5 生成中间文件 local-intermediates-dir

变量定义路径`core/base_rules.mk`
```
intermediates := $(call local-intermediates-dir)
```
## 1.6 函数定义 local-intermediates-dir
方法定义路径`build/core/definitions.mk`

```
# Uses LOCAL_MODULE_CLASS, LOCAL_MODULE, and LOCAL_IS_HOST_MODULE
# to determine the intermediates directory.
#
# $(1): if non-empty, force the intermediates to be COMMON
# $(2): if non-empty, force the intermediates to be for the 2nd arch
define local-intermediates-dir
$(strip \
    $(if $(strip $(LOCAL_MODULE_CLASS)),, \
        $(error $(LOCAL_PATH): LOCAL_MODULE_CLASS not defined before call to local-intermediates-dir)) \
    $(if $(strip $(LOCAL_MODULE)),, \
        $(error $(LOCAL_PATH): LOCAL_MODULE not defined before call to local-intermediates-dir)) \
    $(call intermediates-dir-for,$(LOCAL_MODULE_CLASS),$(LOCAL_MODULE),$(LOCAL_IS_HOST_MODULE),$(1),$(2)) \
)
endef
```


## 2.1 日志输出
##### 2.1.1 target c++
```target C++: libfmjni xx FMRadio/jni/fmr/common.cpp```

##### 2.1.2 target thumb C++
```target thumb C++: libfmjni_32 xx FMRadio/jni/fmr/common.cpp```

##### 2.1.3 target SharedLib
```target SharedLib: libfmjni (obj/SHARED_LIBRARIES/libfmjni_intermediates/LINKED/libfmjni.so)```

##### 2.1.4 target Pack Relocations
```target Pack Relocations: libfmjni_32 (obj_arm/SHARED_LIBRARIES/libfmjni_intermediates/PACKED/libfmjni.so)```

##### 2.1.5 target Symbolic
```target Symbolic: libfmjni_32 (symbols/system/lib/libfmjni.so)```

##### 2.1.6 target Strip
```target Strip: libfmjni_32 (obj_arm/lib/libfmjni.so)```

##### 2.1.7 Install
```Install: system/lib/libfmjni.so```



## 2.1 gcc 定义的路径
```
ifeq ($(strip $(HOST_TOOLCHAIN_PREFIX)),)
HOST_TOOLCHAIN_PREFIX := prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.15-4.8/bin/x86_64-linux-
endif
HOST_CC  := $(HOST_TOOLCHAIN_PREFIX)gcc
HOST_CXX := $(HOST_TOOLCHAIN_PREFIX)g++
HOST_AR  := $(HOST_TOOLCHAIN_PREFIX)ar
```

## 3.1 shared_library 依赖关系


![shared_library 依赖关系](http://upload-images.jianshu.io/upload_images/74811-b2dfa5791a2abcdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
