---
layout: single
title: "NVIDIA vGPU方案解析"
date: 2026-01-29 16:00:00 +0800
categories: 技术 虚拟化
tags: [GPU, NVIDIA, vGPU, VFIO]
excerpt: "NVIDIA vGPU驱动实现了与VFIO框架的核心交互，是连接NVIDIA专有GPU驱动与Linux标准VFIO/MDEV虚拟化框架的适配器角色。"
---

# NVIDIA vGPU方案解析

## NVIDIA vGPU驱动剖析

`NVIDIA-Linux-x86_64-460.73.01-grid-vgpu-kvm`是一个用于Linux KVM虚拟化环境的NVIDIA vGPU驱动程序。这个驱动实现了同VFIO框架的核心交互点：`vfio_device_ops`。也就是说，VFIO MDEV负责注册虚拟化的驱动设备，生成`vfio_device_fd`句柄，进而调用到底层驱动，是连接NVIDIA专有GPU驱动与Linux标准VFIO/MDEV虚拟化框架的**适配器**角色。

### 前端-后端架构

前端的接口是`nvidia_vgpu_vfio`，后端接口是`RM(Resource Manager)`核心资源管理器。两者之间的通信桥梁是一套定义清晰的操作函数集`(Operations)`。

![回调函数机制](/assets/image-20260130105303922.png)



入口函数`nv_vgpu_vfio_init`主要执行两个关键动作：

1. **建立与后端的通信`（nvidia_vgpu_vfio_set_ops）`**：前端驱动调用此函数，并传入一个空的`vgpu_vfio_ops`结构体指针。后端驱动（即`open-gpu-kernel-modules`中的实现）会填充这个结构体，将自己实现的一系列核心功能函数（如`vgpu_create`、`vgpu_delete`等）的地址赋值给它。这一步完成后，前端就拥有了一张可以指挥后端的“API列表”。

2. **向内核注册自身 (`nv_vgpu_probe`)**: 建立通信后，驱动开始与内核交互。当物理GPU被`nvidia`主驱动探测到时，`nv_vgpu_probe`会被调用，它的核心任务是调用`mdev_register_device`，正式向MDEV框架“宣告”：这块物理GPU支持虚拟化，并指定了`vgpu_fops`作为管理其虚拟设备（即vGPU实例）的操作接口。

#### 具体实现

1. 后端接口的获取：`nvidia_vgpu_vfio_set_ops & nvidia_vgpu_vfio_get_ops`,前端通过`nvidia_vgpu_vfio_get_ops`从后端获取了一系列函数指针。这些函数构成了前端操作vGPU所需的所有原语。其中一些查询类的信息会被展示在sysfs中，供管理员查看和选择。

   ```c
   NV_STATUS nvidia_vgpu_vfio_get_ops(rm_vgpu_vfio_ops_t *ops)
   {
       if (strcmp(ops->version_string, NV_VERSION_STRING) != 0)
       {
           ops->version_string = NV_VERSION_STRING;
           return NV_ERR_GENERIC;
       }
   
       ops->vgpu_create = nvidia_vgpu_create; // 请求后端创建一个vGPU实例
       ops->vgpu_delete = nvidia_vgpu_delete; // 请求后端销毁一个vGPU实例
       ops->vgpu_bar_info = nvidia_vgpu_bar_info; // 返回 vGPU 的 PCI BAR 配置信息
       ops->vgpu_start = nvidia_vgpu_start; // 激活vGPU实例，使其可以开始处理GPU操作
       ops->get_types = nvidia_vgpu_get_types; // 返回此物理 GPU 支持的所有 vGPU 类型 ID
       ops->get_name = nvidia_vgpu_get_name; // 返回指定 vGPU 类型的显示名称
       ops->get_description = nvidia_vgpu_get_description; // 返回 vGPU 类型的详细配置信息
       ops->get_instances = nvidia_vgpu_get_instances; // 返回指定 vGPU 类型的实例使用情况
       ops->get_sparse_mmap = nvidia_vgpu_get_sparse_mmap; // 返回 BAR 区域中哪些部分可以直接映射，哪些需要陷入模拟
       ops->update_request = nvidia_vgpu_update_request; // 处理 vGPU 运行时的配置更新和特殊请求
   
       return NV_OK;
   }
   ```

2. **MDEV父设备的注册 (`nv_vgpu_probe`)**

   当GPU物理设备准备就绪时，`nv_vgpu_probe`函数会被执行，它负责将GPU注册为一个可以承载MDEV虚拟设备的“母体”。

   ```c
   static vgpu_vfio_ops_t vgpu_vfio_ops = {
       .probe                   = nv_vgpu_probe,
       .remove                  = nv_vgpu_remove,
       .inject_interrupt        = nv_vgpu_inject_interrupt,
   };
   ```