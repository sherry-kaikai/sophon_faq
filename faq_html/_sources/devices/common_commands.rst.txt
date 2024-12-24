.. 一级标题是 ---
.. 二级标题是 ~~~
.. 三级标题是 ^^^

.. |84&84X| replace:: BM1684、BM1684X
.. |A2| replace:: BM1688、CV186AH

.. _bm-smi介绍: https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/libsophon/guide/html/3_1_bmsmi_description.html#bm-smi
.. _SoC模式内存修改工具: https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/SophonSDK_doc/zh/html/appendix/2_mem_edit_tools.html
.. _查询内存用量: https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/sophon-img/reference/html/1_BM1684X-software.html#id26
.. _proc文件系统介绍: https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/libsophon/guide/html/3_tools.html#proc

.. _BMRuntime手册: https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/tpu-runtime/reference/html/bmrt_test/bmrt_test.html
.. _BMRuntime手册——tpu_model: https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/tpu-runtime/reference/html/bmodel/bmodel.html#tpu-model


设备版本、状态、bmodel模型信息如何获取？
------------------------------------------

.. contents:: 目录


固件版本和SDK版本如何查看？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

基于 |84&84X| 的设备如何查看固件版本、SDK信息？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

答：

- SOPHONSDK版本号对应说明：https://developer.sophgo.com/thread/597.html；
- PCIe模式：  ``cat /proc/bmsophon/card0/versions``； 或 ``ls /opt/sophon``；
- SOC模式： ``bm_version`` ；
- 或使用getBasicInfo工具获取完整环境信息： 执行此命令 ``wget -qO - https://sophon-file.sophon.cn/sophon-prod-s3/drive/24/01/03/14/getBasicInfo.tar.gz | tar -xvzO | bash`` ，执行后环境信息会保存在getBasicInfo_<日期>.html中。


基于 |A2| 的 SoC 设备如何查看固件版本、SDK版本信息？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

答：

- 查看 ``bm_version``；
- 或使用getBasicInfo工具获取完整环境信息： 执行此命令 ``wget -qO - https://sophon-file.sophon.cn/sophon-prod-s3/drive/24/01/03/14/getBasicInfo.tar.gz | tar -xvzO | bash`` ，执行后环境信息会保存在getBasicInfo_<日期>.html中。


如何获取设备状态信息？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

bm-smi等设备管理工具
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

答：

我们提供bm-smi工具、proc文件系统接口和sysfs文件系统接口用于获取设备状态信息。

- bm-smi以界面或者文本的形式显示设备状态信息，如设备的温度、风扇转速等信息；也可使能、禁用或者设置设备的某些功能，如led、ecc等。详细使用方法见 `bm-smi介绍`_ 。
- proc文件系统接口：在/proc 节点下创建设备信息节点，用户通过cat或者编程的方式读取相关节点获取设备温度、版本等信息。 
- sysfs 文件系统接口：用来获取 TPU 的利用率等信息。

.. note:: 
   bm-smi侧重于以界面的形式直观显示设备信息，proc文件系统接口侧重于为用户提供编程获取设备信息的接口。


PCIe设备状态信息常用命令？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


1. 检查PCIe加速卡： ``lspci | grep Sophon`` 
2. 检测驱动是否安装： ``ls /dev/bm*``
3. 读取MCU信息： ``cat /sys/bus/i2c/devices/1-0017/information``
4. 读取处理器温度： ``cat /sys/class/thermal/thermal_zone0/temp``
5. 读取核心板温度： ``cat /sys/class/thermal/thermal_zone1/temp``
6. 读取功耗信息： ``sudo pmbus -d 0 -s 0x50 –i``
7. 查看ION内存： ``bm-smi --opmode=display_memory_detail`` 或者：

.. code-block:: bash

   sudo -i
   watch cat /sys/kernel/debug/ion/bm_npu_heap_dump/summary | head -2
   watch cat /sys/kernel/debug/ion/bm_vpu_heap_dump/summary | head -2
   watch cat /sys/kernel/debug/ion/bm_vpp_heap_dump/summary | head -2

8. 查看VPU、JPU状态： ``watch cat /proc/bmsophon/card0/bmsophon0/media`` ， ``watch cat /proc/bmsophon/card0/bmsophon0/jpu``。 详细使用方法见 `proc文件系统介绍`_ 。



.. _SoC设备状态信息常用命令:
 
SoC设备状态信息常用命令？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- 基于 |84&84X| 的设备：
   - VPU、JPU 硬件使用率 ``sudo watch cat /proc/vpuinfo``， ``sudo watch cat /proc/jpuinfo``
   - 查看ION内存：   

   .. code-block:: bash

      sudo -i
      watch cat /sys/kernel/debug/ion/bm_npu_heap_dump/summary | head -2
      watch cat /sys/kernel/debug/ion/bm_vpu_heap_dump/summary | head -2
      watch cat /sys/kernel/debug/ion/bm_vpp_heap_dump/summary | head -2

- 基于 |A2| 的设备：
   - 获取产品信息：  ``cat /factory/OEMconfig.ini``
   - 查看ION内存: ``sudo watch cat /sys/kernel/debug/ion/cvi_npu_heap_dump/summary | head -2``; ``sduo watch cat /sys/kernel/debug/ion/cvi_vpp_heap_dump/summary | head -2``
   - 读取处理器温度：  ``cat /sys/class/thermal/thermal_zone0/temp``

注意：SoC模式下硬件内存可能会成为瓶颈，SoC模式下内存修改可以参考 `SoC模式内存修改工具`_ 。


bmodel模型信息如何查看？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

tpu_model
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

答：通过tpu_model命令，可以查看bmodel文件的参数信息，可以将多个网络bmodel分解成多个单网络的bmodel，也可以将多个网络的bmodel合并成一个bmodel。
具体请查看： `BMRuntime手册——tpu_model`_ 

bmrt_test
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

答：bmrt_test是基于bmruntime接口实现的对bmodel的正确性和实际运行性能的测试工具，具体请查看： `BMRuntime手册`_
