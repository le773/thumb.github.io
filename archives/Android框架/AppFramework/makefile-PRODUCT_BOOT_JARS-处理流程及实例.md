## 1.  含义

PRODUCT_BOOT_JARS 最终被编译到system/framework，并被添加到BOOTCLASSPATH路径；

----------

###### 2.1 引用makefile

```
PRODUCT_BOOT_JARS += xxx.jar
```
###### 2.2 DEXPREOPT_BOOT_JARS_MODULES

core/dex_preopt.mk

```
DEXPREOPT_BOOT_JARS := $(subst $(space),:,$(PRODUCT_BOOT_JARS))
DEXPREOPT_BOOT_JARS_MODULES := $(PRODUCT_BOOT_JARS)
```
###### 2.3 核心

core/dex_preopt.mk

```
PRODUCT_BOOTCLASSPATH := $(subst $(space),:,$(foreach m,$(DEXPREOPT_BOOT_JARS_MODULES),/system/framework/$(m).jar))
```

core/dex_preopt.mk

```
$(foreach b,$(DEXPREOPT_BOOT_JARS_MODULES),$(eval $(call _dexpreopt-boot-jar-remove-classes.dex,$(b))))
```

###### 2.4 _dexpreopt-boot-jar-remove-classes.dex

core/dex_preopt.mk

```
# Special rules for building stripped boot jars that override java_library.mk rules

# $(1): boot jar module name
define _dexpreopt-boot-jar-remove-classes.dex
_dbj_jar_no_dex := $(DEXPREOPT_BOOT_JAR_DIR_FULL_PATH)/$(1)_nodex.jar
_dbj_src_jar := $(call intermediates-dir-for,JAVA_LIBRARIES,$(1),,COMMON)/javalib.jar

$$(_dbj_jar_no_dex) : $$(_dbj_src_jar) | $(ACP) $(AAPT)
        $$(call copy-file-to-target)
ifneq ($(DEX_PREOPT_DEFAULT),nostripping)
        $$(call dexpreopt-remove-classes.dex,$$@)
endif

_dbj_jar_no_dex :=
_dbj_src_jar :=
endef
```
###### 注释:
> 先编译$$(_dbj_src_jar), 然后对$$(_dbj_jar_no_dex) 做$(ACP) $(AAPT)

###### 2.5 AAPT

core/config.mk

```
AAPT := $(HOST_OUT_EXECUTABLES)/aapt$(HOST_EXECUTABLE_SUFFIX)
```

core/combo/HOST_windows-x86.mk

> HOST_EXECUTABLE_SUFFIX := .exe

###### 2.6 ACP

core/config.mk

```
ACP := $(BUILD_OUT_EXECUTABLES)/acp$(BUILD_EXECUTABLE_SUFFIX)
```

###### 2.7 DEXPREOPT_BOOT_JAR_DIR_FULL_PATH

```
DEXPREOPT_BOOT_JAR_DIR_FULL_PATH := $(DEXPREOPT_PRODUCT_DIR_FULL_PATH)/$(DEXPREOPT_BOOT_JAR_DIR)

DEXPREOPT_PRODUCT_DIR_FULL_PATH := $(PRODUCT_OUT)/dex_bootjars
DEXPREOPT_BOOT_JAR_DIR := system/framework
```

###### 2.8 copy-file-to-target

core/definitions.mk

```
# The -t option to acp and the -p option to cp is
# required for OSX.  OSX has a ridiculous restriction
# where it's an error for a .a file's modification time
# to disagree with an internal timestamp, and this
# macro is used to install .a files (among other things).

# Copy a single file from one place to another,
# preserving permissions and overwriting any existing
# file.
# We disable the "-t" option for acp cannot handle
# high resolution timestamp correctly on file systems like ext4.
# Therefore copy-file-to-target is the same as copy-file-to-new-target.
define copy-file-to-target
@mkdir -p $(dir $@)
$(hide) $(ACP) -fp $< $@
endef
```

###### 2.9 PRODUCT_BOOTCLASSPATH

core/product.mk

```
_product_stash_var_list := $(_product_var_list) \
        PRODUCT_BOOTCLASSPATH \
        ...
```

###### 2.10stash-product-vars

core/product.mk

```
#
# Stash values of the variables in _product_stash_var_list.
# $(1): Renamed prefix
#
define stash-product-vars
$(foreach v,$(_product_stash_var_list), \
        $(eval $(strip $(1))_$(call rot13,$(v)):=$$($$(v))) \
 )
endef
```

system_server jar > framework.jar

#### 总结：
BOOTCLASSPATH中中的jar编译为boot.art和boot.oat

----------

##  实战
###### 3.1 编写Jar包
结构
```
fox
├── Android.mk
└── java
    └── com
        └── android
            └── host
                └── fox
                    └── HelloFox.java
```
3.1.1 Android.mk

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := host-fox // 默认编译到system/framework
#LOCAL_MODULE_TAGS := optional

LOCAL_SRC_FILES := $(call all-java-files-under,java)

include $(BUILD_JAVA_LIBRARY)
```

3.1.2 HelloFox.java
```
package com.android.host.fox;
public class HelloFox {
    public String say() {
        return "Hello, host fox";
    }
}
```

3.1.3 在系统makefile添加模块host-fox
```
# add jars
PRODUCT_PACKAGES += \
     host-fox 
// 添加到Android编译系统

PRODUCT_BOOT_JARS += host-fox // 添加到BOOTCLASSPATH路径
```

结果展示

![host-fox.jpg](http://upload-images.jianshu.io/upload_images/74811-2177390bd6044b96.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
