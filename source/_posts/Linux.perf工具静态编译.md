---
title: Linux perf工具静态编译
date: 2025-04-25 10:53:21
tags: 程序人生
categories: 程序人生
toc: true
---
在嵌入式环境中，有时候相关库比较少，所以需要单独编译静态版本的perf工具去定位问题，这里仅仅作为关键步骤记录，以备后续查阅

<!-- more -->

```patch
# 修改Makefile.config文件
diff --git a/tools/perf/Makefile.config b/tools/perf/Makefile.config
index a92f0f025ec7..87a6e97ff1f8 100644
--- a/tools/perf/Makefile.config
+++ b/tools/perf/Makefile.config
@@ -16,7 +16,7 @@ $(shell printf "" > $(OUTPUT).config-detected)
 detected     = $(shell echo "$(1)=y"       >> $(OUTPUT).config-detected)
 detected_var = $(shell echo "$(1)=$($(1))" >> $(OUTPUT).config-detected)

-CFLAGS := $(EXTRA_CFLAGS) $(filter-out -Wnested-externs,$(EXTRA_WARNINGS))
+CFLAGS := $(EXTRA_CFLAGS) $(filter-out -Wnested-externs,$(EXTRA_WARNINGS)) -static

 include $(srctree)/tools/scripts/Makefile.arch

```

```bash
# 执行编译
cd tools/perf
make clean
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- perf  V=1
```

