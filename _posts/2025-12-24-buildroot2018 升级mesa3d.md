---
title: buildroot2018 升级 mesa3d
date: 2025-12-24 20:40:00 +0800
categories: [Linux]
tags: [Linux,buildroot,mesa3d ]
---
<script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>

<link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.3.1/css/all.css" integrity="sha384-mzrmE5qonljUremFsqc01SB46JvROS7bZs3IO2EmfFsd15uHvIt+Y8vEf7N7fWAU" crossorigin="anonymous">

<p align="right"><i class="fa fa-eye"></i> 阅读次数: <span id="busuanzi_value_page_pv"><i class="fa fa-spinner fa-spin"></i></span></p>

由于某些原因，buildroot-2018.11.4 自带的mesa3d 版本(18.2.4)过低,经过测试 buildroot-2021.02.12 默认的mesa3d版本 (20.3.5) 满足需求，因此尝试将buildroot2021 里面的mesa3d 的软件包移植到buildroot2018里面，让buildroot2018 可以正常编译，支持搞版本的mesa3d。


mesa3d 20.3.5 需要更高版本的libdrm 和meson 。buildroot 2018 默认的版本过低也需要进行升级。


# 一、 升级meson


1：拷贝高版本的meson 文件夹。

2：拷贝高版本的package/pkg-meson.mk  文件。

3：python3 版本也有差异，在mk文件中增加`host-python3-setuptools` 依赖。

4：buildroot 2018 有python-setuptools 包，没有python3-setuptools，因此将高版本buildroot中的python3-setuptools拷贝过来，将setuptools 的版本改成跟buildroot2018 一致即可。

```shell
diff --git a/meson/meson.mk b/meson/meson.mk
index 7e39883..71223e1 100644
--- a/meson/meson.mk
+++ b/meson/meson.mk
@@ -10,7 +10,7 @@ MESON_LICENSE = Apache-2.0
 MESON_LICENSE_FILES = COPYING
 MESON_SETUP_TYPE = setuptools
 
-HOST_MESON_DEPENDENCIES = host-ninja
+HOST_MESON_DEPENDENCIES = host-ninja host-python3-setuptools
 HOST_MESON_NEEDS_HOST_PYTHON = python3
 
 HOST_MESON_TARGET_ENDIAN = $(call qstrip,$(call LOWERCASE,$(BR2_ENDIAN)))

```


# 二、升级libdrm

meson升级好后，libdrm 直接从高版本拷贝到低版本即可。

# 三、升级mesa3d

1：将高版本的mesa3d 拷贝过来。

2：着重关注mesa3d.mk里面的一个依赖项目： host-python3-mako  ，buildroot2018 没有这个包，因此将高版本的 python3-mako 和 python-mako 都拷贝过来。

3：编译发现 host-python3-mako 依赖 host-python3-markupsafe  因此将python3-markupsafe和python-markupsafe 拷贝过来，在`python3-mako.mk` 中添加一行 `HOST_PYTHON3_MAKO_DEPENDENCIES = host-python3-markupsafe`




# 四、总结
完成上述步骤后，buildrroot2018 即可编译通过高版本的mesa2d3d。
`
