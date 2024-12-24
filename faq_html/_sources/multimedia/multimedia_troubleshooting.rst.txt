多媒体问题排查手册
^^^^^^^^^^^^^^^^^^



.. contents:: 目录


名词定义
============

任务

-  拉流

   -  拉流是指从视频服务器或者流媒体服务器上获取视频数据的过程。拉流本身不涉及解码。主要涉及网络和CPU。

-  解码

   -  解码是将压缩的视频数据转换回可观看的视频画面的过程。简而言之，它是将编码的视频数据（通常体积较小，便于传输）转换为可以在屏幕上显示的视频帧。主要涉及VPU。

-  编码

   -  编码是将视频画面转换成压缩的数据格式的过程。这样做可以减小视频文件的大小，便于存储和传输。主要涉及VPU。

-  转码

   -  解码再编码。主要涉及VPU。

-  推流

   -  推流是指将本地的视频数据发送到流媒体服务器或者其他接收端的过程。推流本身不涉及编码。主要涉及网络和CPU。

模式

-  命令行

   -  在命令行中使用ffmpeg完成某些任务。

-  C++开发

   -  使用FFmpeg / OpenCV / SAIL 提供的接口开发程序。

-  Python开发

   -  使用SAIL / OpenCV 提供的接口开发程序。

FFmpeg问题
==============

ffmpeg拉流问题
--------------

1. rtsp method DESCRIBE failed 401 Unauthorized
   RTSP认证错误。

    1. 首先检查用户名或者密码是否正确。
    2. 检查用户名和密码是否含有特殊字符。对于 ``rtsp://{user}:{passwd}@{ip}:{port}/{path}`` 形式的RTSP地址，如果user和peasswd中含有 "`+`""`#`""`%`""`@`"等字符，需要把特殊字符用URL Encoding替换。


2. method SETUP failed: 461 Unsuported transport
   使用了不支持的传输协议。


    1. 测试 `ffmpeg -rtsp_transport tcp -i <rtsp://xxx>` 。如果失败，说明该RTSP只支持UDP，不支持TCP。
    2. 测试 `ffmpeg -rtsp_transport udp -i <rtsp://xxx>` 。如果失败，说明该RTSP只支持TCP，不支持UDP。

3. connect to xxx failed: Connection refused
   网络问题


    1. 检查RTSP对应的推流服务器是否开启
    2. ping RTSP的IP，是否能连通
    3. 检查ip、网关等是否正确配置

4. 确认该RTSP是否支持被多个客户端拉流
5. 拉流失败的保底排查方案 

    1. 如果上面的问题都排查过了，大概率就是网络问题了，而非ffmpeg或者设备硬件问题。 
    2. 可以用公版ffmpeg拉流。 `Index of /ffmpeg/old-releases (johnvansickle.com) <https://www.johnvansickle.com/ffmpeg/old-releases/>`_

ffmpeg解码问题
--------------

ffmpeg C++开发问题
~~~~~~~~~~~~~~~~~~

.. 1. VPU_DecGetOutputInfo decode fail framdIdx xxx error(0x00000000) reason(0x00400000), reasonExt(0x00000000)某一帧解码失败，一般是这一帧不合法，原因一般在：

#. VPU_DecGetOutputInfo decode fail framdIdx xxx error 某一帧解码失败，一般是这一帧不合法，原因一般在：

    1. rtsp流默认为udp协议，有丢包。命令中解码参数加上 -rtsp_transport tcp。

    2. 网络带宽较小，多路解码带宽占满，造成丢包。

    3. 设备解码能力到达上限了。这里说的"上限"，未必是VPU本身的硬件性能到上限了。比如，有可能是上层代码没有合理利用CPU等资源，导致VPU不能完全发挥性能。比如，如果使用ffmpeg命令行执行RTSP拉流解码+编码，而没有设置硬编码，因为A53编码较慢，编码阻塞，进而影响解码的调度，出现数据丢失。这时修改命令调用VPU硬编码可以解决。  

    4. 推流不稳定。可以用VLC播放，观察是否有花屏。

#. Invalid NAL unit 0, skipping / missing picture in access unit /Failed to DEC_PIC_HDR(ERROR REASON: 00005000) error code is 0x1 / InstIdx 0: BMVidDecSeqInitW5 failed Error code is 0x1 / Invalid NAL unit size / Error splitting the input into NAL units 
    
    1. 检查解码器是否匹配。比如，是否输入文件为h265编码而指定了-c:v h264_bm，或者输入文件是mpeg4编码而指定了-c:v h264_bm

#. no frame / missing picture in access unit with size / data partitioning is not implemented 
    
    1. 检查解码器是否匹配。比如，是否输入文件为h264编码而指定了-c:v hevc_bm

#. bm_alloc_gmem failed 
    
    1. 内存满了。一般是VPU内存满了。可以考虑使用压缩格式output_format 101，或者适当减小extra_frame_buffer_num，或者修改内存布局。 
    2. 句柄到上限了。ulimit -n 查看当前环境句柄上限，如果是1024很可能不够用。ulimit -n {num} 修改上限，一般65536够用了。

#. ``avcodec_receive_frame`` 返回错误码 -11 
    
    1. -11对应EAGAIN，Resource temporarily unavailable， https://ffmpeg.org/doxygen/trunk/error_8c_source.html 
    2. 在ffmpeg中，发生在解码阶段时，表示送入的pkt不够解出一帧frame出来，需要继续send 
    3. 解决方法是用while继续send pakcet


ffmpeg 命令行问题
~~~~~~~~~~~~~~~~~

a. 基本测试命令 
    

.. code-block:: bash
    
    #1. 网络流
    ffmpeg -rtsp_transport tcp -i rtsp://xxx -f rawvideo -y /dev/null
    
    ffmpeg -rtsp_transport udp -i rtsp://xxx -f rawvideo -y /dev/null
    
    ffmpeg -rtsp_transport tcp -extra_frame_buffer_num 5 -i rtsp://xxx -c:v h264_bm -b:v 3M -vframes 500 out.mp4

    #2. 本地视频    
    ffmpeg -i {input} -f rawvideo -y /dev/null
    
    ffmpeg -extra_frame_buffer_num 5 -i {input} -c:v h264_bm -b:v 3M {output}

b. RTSP拉流解码卡住，不报错 

    #. 多见于大华、海康的RTSP 
    #. 如果RTSP的地址有些特殊符号，要注意命令行解析字符串的时候可能出错，解决方法是给RTSP地址加上引号 ``"rtsp://..."``

c. 解码失败的保底排查方案 

    1. 保存原始码流，拿到本地分析码流。有以下两种保存方式。 
    2. ``ffmpeg -rtsp_transport tcp -i rtsp://xxx -c copy -vframes 500 saved.264`` 
    3. ``sudo echo "0 0 1000 0" > /proc/vpuinfo``   参数1：0 core idx；参数2：0 instance idx；参数3：输入num（保存的帧数）；参数4：输出num（保存的yuv）；执行编解码，然后dump的文件在/data/core_%dinst%d_stream%d.bin

ffmpeg编码/转码问题
-------------------

1. ffmpeg命令行转码卡住、阻塞 
    1. 出现阻塞现象时，命令一般是 ``ffmpeg -i {input} -c:v h264_bm -b:v 3M {output}`` 
    2. VPU内部维护了一个ring buffer供解码和编码使用。编码一般需要若干个缓存帧。如果设置buffer num过小，buffer被编码全部占用而不释放，解码和编码就会阻塞。 
    3. 解决方法是，通过 ``-extra_frame_buffer_num {buffer_num}`` 增大缓存buffer。
2. ERROR! Invalid pic data 
    1. 送给编码器的frame是空的。 
    2. 对于命令行，比如从rawvideo（一般是从摄像头或本地文件）读数据，数据在系统内存上，VPU默认从设备内存读数据，所以读不到有效数据，改成is_dma_buffer 0可以解决。 
    3. 对于C++开发，如果is_dma_buffer=1，检查设备内存（data[4-7]），如果is_dma_buffer=0，检查系统内存（data[0-3]）
3. 编码得到的视频播放时回退或者帧顺序不对 
    1. :ref:`ffmpeg中做图像格式/大小变换导致视频播放时回退或者顺序不对的情况处理办法 <ffmpeg中做图像格式/大小变换导致视频播放时回退或者顺序不对的情况处理办法>`
4. ffmpeg **C++开发** ，编码绿屏、花屏 
    1. 编码的核心数据结构是AVFrame，sophon-ffmpeg对公版AVFrame做了拓展，见文档 `2.3.7. AVFrame特殊定义说明 – Multimedia v23.10.01 文档 <https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/sophon-mw/guide/html/guide/Multimedia_Guide_zh.html#avframe>`_ 。简单来说，data[0-3]保存了系统内存地址，data[4-7]保存了设备内存地址。一般来说，VPU用的是data[4-7]，不会需要用系统内存。
    2. 如果编码出错，可以排查以下几点：
        1. 确认is_dma_buffer的值，是0还是1。默认为1，表示用设备内存编码，设置为0表示从系统内存编码。
        2. 如果is_dma_buffer==1，检查data[4-7]。主要检查两点：1.内存地址是否正确。对于84&X，VPU要求输入在VPU heap上，也就是heap 2 / DDR 1，对应的addr 一般为0x3_8000_0000~0x4_0000_0000。 2. 内存上的像素值是否正确。具体方法是，把对应plane的内存d2s，然后看像素值是否和期望的画面相符，可以gdb也可以fwrite/ofstream。
        3. 如果is_dma_buffer==0，检查data[0-3]内存上的像素值。
        4. 如果上述检查都没问题，但编码出错，还可以考虑是否编码结束之前内存被释放了。

5. ffmpeg **命令行** ，解码后保存视频或者图片，画面是 **绿屏** 或者 **花屏** 

    1. 整个过程是先解码、再编码，解码后的内存没有被编码器正确地读取，这是该问题的根本原因。 
    2. “正确地读取” = **( 编码器和解码器都使用设备内存 || 编码器和解码器都使用设备内存 || 解码器和编码器使用的内存不一致但做了内存拷贝)**
    3. 举例 ``ffmpeg -rtsp_transport tcp -i  "rtsp://"  -vframes 1 frame.jpg``
        #. ffmpeg默认调用VPU硬件解码视频，且默认zero_copy=1，所以解码后的数据  **只** 在VPU  **设备** 内存上。
        #. 输出为jpeg，编码器默认用软编码，从  **系统** 内存读数据。如果这块内存是全0，由于YUV和RGB的色彩对应关系，画面表现为绿屏。如果这块内存最近被用过，可能是一块非零的值，画面表现为花屏。
        #. 解决方法： 
            1. 调用硬件编码器jpeg_bm，且因为JPU直接读VPU的数据可能有问题，所以要通过zero_copy选项做一次数据拷贝，并通过is_dma_buffer选项指明JPU的输入来源。
                - ``ffmpeg -rtsp_transport tcp -zero_copy 0 -i "rtsp://" -c:v jpeg_bm -is_dma_buffer 0 -vframes 1 frame.jpg``

            2. 也可以选择调用软编码，那么只需要通过zero_copy做一次拷贝，让CPU能读到数据即可。这个方案缺点是，编码性能较低。

                - ``ffmpeg -rtsp_transport tcp -zero_copy 0 -i "rtsp://" -vframes 1 frame.jpg``

ffmpeg推流问题
--------------

1. 流服务器如何搭建 
    1. Mediamtx，开源、轻量、易上手，适合简单验证，但性能和稳定性略差。 
    2. Live555，稳定，适合性能测试，或者测试业务程序全流程。 
    3. ZLMediaKit

ffmpeg综合问题
--------------

1. 拉流解码，一开始正常，一段时间后报错 
    1. 不管用ffmpeg还是opencv拉流解码，每两次拉流read()的时间间隔一定要小于两帧的时间间隔（25FPS -> 40ms） 
    2. 如果程序是串行的，且推理+后处理>40ms，那么一定会导致解码出错。解决方法是把解码用一个单独的线程并行。
2. 单独解码、单独编码时，速度都能达到标称FPS或者路数，但编解码一起就达不到 
    1. VPU DDR带宽不够了
3. 性能测试，达不到产品手册标称值 
    1. 测试过程是否有报错信息，如果有，根据报错信息修改测试流程。 
    2. 确认性能测试程序或者脚本是否开启了多线程，单线程无法充分发挥硬件性能，峰值性能需要在多线程下测得。 
    3. 确认DDR interleave模式，不同模式下编解码性能有明显差异。

BMCV问题
============

1. send api error 

    1. 硬件hang了，一般是TPU

2. bm_malloc_device_byte_heap error … bm_alloc_gmem failed, dev_id=xxx, size = 0x5eec00

   1. 这个错误一般是发生在给bm_image 分配内存时，显式或隐式地调用了alloc_dev_mem，所指定的heap可用内存不足了。一般是vpp heap，也就是 heap 1。 
   2. 可以运行过程中观察 ``sudo cat /sys/kernel/debug/ion/bm_vpp_heap_dump/summary | head -2`` ,usage 是否接近或者达到100% 
   3. size = 0x5eec0，报错中的size可以帮助定位具体是哪里分配内存失败了。 ``0x5eec00=6,220,800=1920*1080*3`` ，可以推断这是在分配一个类似于100p BGR_planar的图像内存； ``2F7600=3,110,400=1920*1080*1.5``，1080p的YUV420图像。 
   4. 解决方案主要是两方面：   1. 修改内存布局，见SDK手册； 2. 优化程序，特别是申请和释放内存相关的操作，长生命周期的内存，在程序开始统一申请，短生命周期的内存，及时释放。

3. [BMCV][error] bm1684x vpp frame_idx 0, width or height abnormal,input[frame_idx].width 1920,input[frame_idx].height 1080,src_crop_rect[frame_idx].crop_w 34,src_crop_rect[frame_idx].crop_h 6,output[frame_idx].width 224, output[frame_idx].height 224,dst_crop_rect … 
    
    1. 输入尺寸不符合要求。1684x vpp最小支持8x8。

4. [BMCV][error] src device memory start address not aligned, src_addr:4402c6a9f, bmcv_1684_vpp_internal.cpp: check_vpp_input_param 
    
    1. 错误原因是input bm_image的设备内存起始地址没有对齐。vpp对起始地址有对齐要求。bmcv接口得到的一般是对齐的。可以用copyto一下，后续用新得到的bm_image。

5. 使用bmcv的vpp_convert vpp_basic等接口转换出来的图片，与opencv.resize得到的图片，色彩有差异 
    
    1. 原因在于bmcv默认的色彩转换矩阵csc mastrix与OpenCV不同。可以在vpp_basic等接口指定csc类型，或者自定义csc matrix

OpenCV问题
==============

OpenCV拉流、解码
----------------

1. maybe grab ends normally, retry count = 513 
    1. 如果是文件一般是文件播放到末尾需调用VidoeCapture.release后重新VideoCapture.open。 
    2. 如果是拉流，先确认两次read的时间间隔要小于帧间隔，比如对于25FPS的流，两次read小于40ms。 
    3. 推流不稳定或者网络环境不稳定。可以用 ``ffmpeg -rtsp_transport tcp -i <rtsp://xxx> -f rawvideo -y /dev/null`` 长时间拉流测试，看是否会有报错。

OpenCV图像处理
--------------

1. “terminate called after throwing an instance of ‘cv::Exception’ what(): OpenCV(4.1.0) …… matrix.cpp:452: error: (-215:Assertion failed) u != 0 in function ‘creat’” 
    1. 句柄数超过系统限制，ulimit -n。查看当前进程所使用的句柄数： ``lsof -n|awk '{print $2}'|sort|uniq -c|sort -nr|more`` 
    2. vpp内存不足。 ``sudo cat /sys/kernel/debug/ion/bm_vpp_heap_dump/summary | head -2``
2. “Memory allocated by user, no device memory assigned. Not support BMCV!” 
    1. Mat只分配了系统内存，没有分配设备内存 
    2. 如果是8UC3，需要s2d 
    3. 如果是8UC1，请参考  :ref:`8UC1问题FAQ <8UC1-general>`
    

SAIL问题
============

sail.Decoder解码
----------------

#. VPU_DecGetOutputInfo decode fail framdIdx xxx error(0x00000000) reason(0x00400000), reasonExt(0x00000000)

    1. sail.Decoder是单线程的，没有独立的解码进程。如果程序中解码后还有预处理、推理、后处理等流程，且耗时较长（>1s/FPS），会导致Decoder.read()不及时，丢失数据，最终解码失败。
    2. 推荐使用sail.MultiDecoder替换sail.Decoder，MultiDecoder里面每个channel会单独开一个解码线程，解码线程把解码后的BMImage放到队列里，每次read是从队列里直接取BMImage。

#. bm1684x vpp When compressing the format, cropw requires 16 alignment, croph requires 4 alignment, and start_x requires 32 alignment.start_y requires 2 alignment

    1. 这个问题发生在使用了压缩格式时，即 `sail.Decoder(path, compressed=True, dev_id)`。
    2. 如果是24年4月之前的sail，建议升级到最新版，应该会解决这个问题。如果不方便升级，可以把True改成False。

#. 拉流延迟较高

    1. 推流用RTSP，比RTMP延迟低 
    2. sail.set_decoder_env(“probesize”, “4096”) 
    3. sail.set_decoder_env(“analyzeduration”, “1000000”)

sail.Encoder编码
----------------

1. 编码得到的视频，播放速度变快/看起来像加速了，或者总时长变短 
    1. Encoder内部用子线程编码，video_write()只负责送入队列。 
    2. 送入队列的动作，有可能失败（比如队列满）。 
    3. 如果不检查返回值，直接继续送下一帧，会导致很多失败的帧被丢掉，而编码的FPS不变，从最终结果上来看，像是视频加速了。 *假设一共有100个图像，编号1~100，我们希望编码一个25FPS的视频。假设write时所有偶数帧失败，共发生了50次失败，还剩50帧。   本来应该100frames / 25FPS = 4s播放的画面，2s就播完了，所以看起来像2倍速了。*


.. code-block:: python

    # 不推荐的写法：
    while True:
        img = decoder.read()
        encoder.video_write(img)
    
    # 推荐的写法：
    while True:
        img = decoder.read()
        ret = -1
        while (ret):
            ret = encoder.video_write(img)

2. 编码得到的视频，总帧数比video_write()的次数少 
    1. Encoder内部用子线程编码，video_write()只负责送入队列。 
    2. 子线程在第一次video_write()时启动，启动需要短暂的时间。 
    3. 如果编码帧数较少、write速度较快、Encoder很快退出，会导致有一部分BMImage来不及编码。

Related Resources
=================

`BMLIB开发参考手册 – BMLIB-REFERENCE v23.10.01 文档 (sophgo.com) <https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/libsophon/reference/html/index.html>`_

`BMCV 开发参考手册 – BMCV v23.10.01 文档 (sophgo.com) <https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/bmcv/reference/html/index.html>`_

`多媒体开发参考手册 – Multimedia v23.10.01 文档 (sophgo.com) <https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/sophon-mw/guide/html/index.html>`_

`MULTIMEDIA使用手册 – Multimedia Manual 0.9.0 文档 (sophgo.com) <https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/sophon-mw/manual/html/index.html>`_

`多媒体客户常见问题手册 – Multimedia FAQ v23.10.01 文档 (sophgo.com) <https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/sophon-mw/faq/html/index.html>`_

`SOPHON-SAIL 用户手册 – SOPHON-SAIL v23.10.01 文档 (sophgo.com) <https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/sophon-sail/docs/zh/html/index.html>`_
