# Video Buffer
## Overview
* Data buffer allocation
    * Handle buffer allocation for video modules 
* Data channel management
    * Handle data transferring and communication policies

## Scenario
<ol>
  <li>Write / Read data 
    <ol>
      <li>Write @ 30 fps, read @ 30 fps (write read same speed).</li>
        <ol>
          <li>Transmitting and receiving rate perfectly match, no data loss.</li>
          <li>For some reason, Reader decides to drop specific frames at a given time.</li>
        </ol>
      <li>Write @ 30 fps, read @ 15 fps (write fast read slow)
        <ol>
          <li>Readers aware of slower receiving rate, drops one buffer for every two buffer received.</li>
          <li>Writer is to be suspend when no empty buffer available, Reader can receive buffer without data loss.</li>
          <li>Writer returns with error when no empty buffer available.</li>
          <li>Writer forcibly releases multiple buffers, so that it could always request a new buffer.</li>
        </ol>
      </li>
      <li>Write @ 30 fps, read @ 60 fps (write slow read fast)
        <ol>
          <li>Reader is to be suspend when no written buffer available.</li>
        </ol>
      </li>
    </ol>
  </li>
  <li>A tuning module that could analyze images and calculate appropriate configurations for input stage reads buffer at a much slower speed(e.g. 5 fps), should not block Reader from receiving incoming data.</li>
  <li>Writer dispatches a buffer down to multiple Readers. The buffer is released back to buffer pool after all Readers release the buffer.</li>
  <li>(Optional) After Writer fills a buffer, it transfer the buffer at delay time instead of transfer immediately while still writing to the next buffer.</li>
  <li>Directly transfer the buffer from Writer to Reader without copy its content to prevent CPU from redundant data copy.</li>
  <li>Able to suspend modules at any time.</li>
  <li>Able to inspect memory usage at any time.</li>
  <li>Able to request a memory space of certain size from buffer allocator.</li>
  <li>Solutions that is free of memory fragmentation (Memory fragmentation means the memory we allocate is not continuous).</li>
  <li>Able to retrieve data (images/frames) from user space and pass it back to kernel space (e.g. encoder).</li>
  <li>(Optional) If a hook function is registered to a Writer or Reader, the hook function will be executed right before Writer/Reader access the buffer.</li>
  <li>(Advanced) Rowan is an advanced engineer who designed an event-triggered system. This system has a event detector, and preserves the past 20 input frames; if the event happens, the system will save the pre-recorded
frames as well as the following 30 frames to a storage device.</li>
  <li>Sam develops an event detector, which get a YUV image and analyze
it. However, this module consumes a lot CPU resource so that Sam
want to trigger this module at 1 fps. That is, he thinks this module
only wake-up at 1 fps and prevent unnecessary wake-up for content
switch issues.</li>
  <li>Tina is a video application engineer. She configure three video components, input path, video device and video encoder and gets an encoded
bit-stream. She first creates three components and configure them separately. After she has configured the modules, she directly start input path. She has some imaginations that the module start working immediately as she start this module. It periodically writes output images to a tentative buffer (or write to null space). Then she binds the input path and the video device, the output image of input path flows to video devices. However, because she haven't start video device, those images will be blocked at the door of video device. Next, the video device start to take the images as Tina start video device and write its output to a tentative buffer. Follow the similar step, the images are encoded and store into bit-stream buffers.</li>
</ol>

## Goals and Non-Goals
### Goals
1. Manage video buffer, including allocation and release.
2. Support video buffer memory configuration by static user input.
3. Provides video buffer usage.
4. Manage data transferring between single transmit unit and single receive unit (1-to-1).
5. Manage data transferring between single transmit unit and multiple receive unit (1-to-many).
6. Support hook function entry for transmit unit.
7. Support hook function entry for receive unit.
8. Support ==buffered== write function for transmit unit.
9. Support ==non-buffered== ==blocking== write function for transmit unit.
10. Support ==non-buffered== ==non-blocking== write function for transmit unit.
11. Support ==buffered== ==blocking== read function for receive unit.
12. Support ==buffered== ==non-blocking== read function for receive unit.
#### Notes
1. Buffered v.s. Non-buffered
    * Non-buffered I/O simply means that don't use any buffer while reading or writing. (read / write char by char)
    * Buffered I/O simply means that we are not dealing with single char but a block of chars.
2. Blocking v.s. Non-blocking
    * Blocking means that control does not return to the application until the I/O is complete, also called synchronous.
    * Non-blocking will return immediately without waiting I/O to complete, also called asynchronous.
### Non-Goals
1. Does not support buffer management for modules irrelevant to Augentix Video Platform.
2. Does not support data channel for modules irrelevant to Augentix Video Platform.
3. Does not support data transferring between multiple transmitting unit that may write to the same buffer.
4. Does not support runtime re-configuration of video buffer pool.
5. Does not support scheduling of receive units; i.e. when a buffer is dispatched to multiple receive unit, the order of receiving is not determined by Video Buffer subsystem - it should be managed by either consumer drivers or kernel scheduler.

## Video Buffer Management
![HiSilicon video buffer system](https://i.imgur.com/D4aIxQd.png)
This figure shows the framework of video buffer management of HiSilicon. Typically there are two independent parts, **buffer pool allocation** and **buffer transfer**.

- In the beginning, user specifies two buffer *pool A* and *pool B* where each pool contains a number of video buffer with a consistent size. The VB modules allocate blocks from a continuous memory spaces.
- First, *VI* device asks *VB* for a empty buffer *B~m~*, and then writes data to this buffer. 
- After buffer was filled, it transfer this buffer to *VPSS* module. 
- *VPSS* module requests three empty buffers call *B~i~*, *B~j~*, *B~k~* and fill data to them. 
- After writing is finished, *VPSS* release buffer *B~m~* and transfer *B~i~*, *B~j~*, and *B~k~* to module *VENC*, *VDA*, and *VO* respectively.
- Finally the module will release *B~i~*, *B~j~*, and *B~k~* after they finish their jobs.

There's an important action order, *VPSS* first requests three empty buffers, writing these three buffers from *B~m~*, and then **release *B~m~*** while writing finish.

## Inter-communication
![](https://i.imgur.com/hUBOoY1.png =x350)
This figure shows the concept of binder which is responsible for inter-communications. The concept of binder is very similar to socket concept (ip address + port) of computer networking.
- The squares are writer ports
- The circles are reader ports
- One writer port can transmit data to multiple reader ports
- One reader port can only receive date from one writer port in a time

Note that every hardware module can be replaced by a piece of software code in user space. Therefore, Binder system should have capability of being controlled from user space.


