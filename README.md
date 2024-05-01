
//--------------------------------------------------------------------------------------------------
FFMpeg плагин-обёртка для драйвера поддерживающего интерфейс v4l2 mem2mem, аппаратного кодера Cedar, который есть в микроконтроллерах  Allwinner, в частности в v3s.  
Я создал этот драйвер с целью переноса функционала в пространство v4l2 mem2mem и ускорения  
работы видеоконвейера.  
 
В чём проблема?  
После того как я запустил h264 кодирование видеопотока с камеры, используя это решение:  
Прект драйвера аппаратного кодера Cedar:  
https://github.com/noblock/sunxi-cedar-mainline  
Он работает с плагином-обёрткой: cedrus264, который реализован в этом проекте FFMpeg:  
https://github.com/stulluk/FFmpeg-Cedrus  
я обнаружил, что максимальная скорость кодирования на Allwinner v3s, с разрешением 1080p,  
не превышает 12fps.  
Я проанализировал работу конвейера обработки видеопотока и обнаружил место в котором   
происходит задержка. Это копирование буфера с кадром из пространства драйвера устройства  
ввода, в пространство драйвера кодера Cedar, которое происходит в обёртке cedrus264.  
Эту проблему решили во фреймворке v4l2 mem2mem, добавив возможность экспорта-импорта буфера   
из одного драйвера в другой.  
Для работы с кодеками v4l2 mem2mem в FFMpeg есть плагин: h264_v4l2m2m, но он не поддерживает  
экспорт-импорт буферов. Вот я и решил создать свой.  

За основу я взял этот прект драйвера аппаратного кодера Cedar:  
https://github.com/noblock/sunxi-cedar-mainline  
Он работает с плагином-обёрткой: cedrus264, который реализован в этом проекте FFMpeg:  
https://github.com/stulluk/FFmpeg-Cedrus  
И драйвер vdpau Cedrus из Linux mainline:  
https://elixir.bootlin.com/linux/v6.3.2/source/drivers/staging/media/sunxi/cedrus  
в котором реализован интерфейс v4l2 mem2mem, но  
он имеет функционал только декодера. Я портировал функционал драйвера h264 кодера Cedar в  
пространство драйвера vdpau Cedrus. Таким образом я получил драйвер аппаратного h264 кодера  
Cedar с интерфейсом фреймворка v4l2 mem2mem.  

Что нового?  
Плагин устройства ввода:  
вместо v4l2, я создал v4l2m2m_exp, в котором производится экспорт буферов с полученными  
кадрами от камеры, вместо копирования данных.  
Добавил новую опцию:  
-num_capture_buffers - количество буферов для работы конвейера, запрашиваемое у системы.  
Раньше по умолчанию это количество было равно 256. В Allwinner v3s для работы видеоконвейера  
я выделил 16мб и когда в плагине v4l2 запрашиваются 256 буферов, они занимают всю память,  
что приводит к возникновению ошиби. Теперь для нормальной работы достаточно двух буферов.  
-num_capture_buffers 2  
 
Плагин-обёртка кодека v4l2 mem2mem:   
вместо h264_v4l2m2m, я создал h264_v4l2m2m_exp, в котором производится импорт буферов с   
полученными кадрами от камеры, вместо копирования данных.   

Сборка:  
./configure   --prefix=/usr" --arch=arm --target-os=linux  --cross-prefix=$TOOLCHAIN  --extra-ldflags="-Lusr/lib" --extra-cflags="-I/usr/include" --enable-shared --enable-gpl   

make && make install  

Пример использования для udp стрима:  
ffmpeg -y -hide_banner -f v4l2m2m_exp -num_capture_buffers 2 -r 30 -s 1920x1080 -i /dev/video0 -c:v h264_v4l2m2m_exp -num_output_buffers 2 -num_capture_buffers 2 -f mpegts 'udp://192.168.0.101:5333?pkt_size=1024&buffer_size=65535'  

Этот вариант легко работает на скорости 30fps.  

//--------------------------------------------------------------------------------------------------  
FFMpeg is a wrapper plugin for the driver supporting the v4l2 mem2mem interface, the Cedar hardware encoder, which is found in Allwinner microcontrollers, in particular in v3s.  
I created this driver with the goal of transferring functionality to the v4l2 mem2mem space and speeding up
operation of the video conveyor.  
 
What is the problem?  
After I started h264 encoding the video stream from the camera using this solution:  
Cedar hardware encoder driver project:  
https://github.com/noblock/sunxi-cedar-mainline  
It works with a wrapper plugin: cedrus264, which is implemented in this FFMpeg project:  
https://github.com/stulluk/FFmpeg-Cedrus  
I found that the maximum encoding speed on Allwinner v3s, with a resolution of 1080p,  
does not exceed 12fps.  
I analyzed the operation of the video stream processing pipeline and found a place where  
there is a delay. This is copying a buffer with a frame from the device driver space  
input, into the Cedar encoder driver space, which occurs in the cedrus264 wrapper.  
This problem was solved in the v4l2 mem2mem framework, adding the ability to export-import a buffer  
from one driver to another.  
To work with v4l2 mem2mem codecs, FFMpeg has a plugin: h264_v4l2m2m, but it does not support  
export-import of buffers. So I decided to create my own.  

I took this Cedar hardware encoder driver project as a basis:  
https://github.com/noblock/sunxi-cedar-mainline  
It works with a wrapper plugin: cedrus264, which is implemented in this FFMpeg project:  
https://github.com/stulluk/FFmpeg-Cedrus  
And the vdpau Cedrus driver from Linux mainline:  
https://elixir.bootlin.com/linux/v6.3.2/source/drivers/staging/media/sunxi/cedrus  
which implements the v4l2 mem2mem interface, but  
it has only decoder functionality. I ported the Cedar encoder h264 driver functionality to  
vdpau Cedrus driver space. This way I got a hardware h264 encoder driver  
Cedar with v4l2 mem2mem framework interface.  

What's new?  
Input device plugin:  
instead of v4l2, I created v4l2m2m_exp, which exports buffers with the received  
frames from the camera, instead of copying data.  
Added a new option:  
-num_capture_buffers - the number of buffers for pipeline operation requested from the system.  
Previously, by default this number was 256. In Allwinner v3s, for the video pipeline to work  
I allocated 16MB and when 256 buffers are requested in the v4l2 plugin, they take up all the memory,  
which results in an error. Now two buffers are enough for normal operation.  
-num_capture_buffers 2  
 
V4l2 mem2mem codec wrapper plugin:  
instead of h264_v4l2m2m, I created h264_v4l2m2m_exp, which imports buffers from  
received frames from the camera, instead of copying data.  

Assembly:  
./configure --prefix=/usr" --arch=arm --target-os=linux --cross-prefix=$TOOLCHAIN --extra-ldflags="-Lusr/lib" --extra-cflags="- I/usr/include" --enable-shared --enable-gpl  

make && make install  

Example of use for udp stream:  
ffmpeg -y -hide_banner -f v4l2m2m_exp -num_capture_buffers 2 -r 30 -s 1920x1080 -i /dev/video0 -c:v h264_v4l2m2m_exp -num_output_buffers 2 -num_capture_buffers 2 -f mpegts 'udp://192.168.0.101 :5333?pkt_size= 1024&buffer_size=65535'  

This option works easily at 30fps.  



//--------------------------------------------------------------------------------------------------  
FFmpeg README
=============

FFmpeg is a collection of libraries and tools to process multimedia content
such as audio, video, subtitles and related metadata.

## Libraries

* `libavcodec` provides implementation of a wider range of codecs.
* `libavformat` implements streaming protocols, container formats and basic I/O access.
* `libavutil` includes hashers, decompressors and miscellaneous utility functions.
* `libavfilter` provides a mean to alter decoded Audio and Video through chain of filters.
* `libavdevice` provides an abstraction to access capture and playback devices.
* `libswresample` implements audio mixing and resampling routines.
* `libswscale` implements color conversion and scaling routines.

## Tools

* [ffmpeg](https://ffmpeg.org/ffmpeg.html) is a command line toolbox to
  manipulate, convert and stream multimedia content.
* [ffplay](https://ffmpeg.org/ffplay.html) is a minimalistic multimedia player.
* [ffprobe](https://ffmpeg.org/ffprobe.html) is a simple analysis tool to inspect
  multimedia content.
* Additional small tools such as `aviocat`, `ismindex` and `qt-faststart`.

## Documentation

The offline documentation is available in the **doc/** directory.

The online documentation is available in the main [website](https://ffmpeg.org)
and in the [wiki](https://trac.ffmpeg.org).

### Examples

Coding examples are available in the **doc/examples** directory.

## License

FFmpeg codebase is mainly LGPL-licensed with optional components licensed under
GPL. Please refer to the LICENSE file for detailed information.

## Contributing

Patches should be submitted to the ffmpeg-devel mailing list using
`git format-patch` or `git send-email`. Github pull requests should be
avoided because they are not part of our review process and will be ignored.
