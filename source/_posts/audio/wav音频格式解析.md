---
title: wav音频格式解析
date: 2025-06-09 10:23:23
tags:
    - audio
categories:
    - audio

---

> 参考：
>
> - https://www.cnblogs.com/douzujun/p/10600793.html

# WAV格式专业名词

wav是微软开发的一种音频文件格式，wav文件格式是无损音频文件格式，相对于其他音频格式文件数据是没有经过压缩的，通常文件也相对比较大些。

> 支持多种音频数字，取样频率和声道，标准格式化的WAV文件和CD格式一样，也是44.1K的取样频率，16位量化数字，因此在声音文件质量和CD相差无几！ 
> 通常使用**三个参数来表示声音**，**量化位数，取样频率和[采样点](http://link.zhihu.com/?target=http%3A//baike.baidu.com/item/%E9%87%87%E6%A0%B7%E7%82%B9)振幅**。**量化位数**分为**8位，16位，24位**三种，**声道**有**单声道和[立体声](http://link.zhihu.com/?target=http%3A//baike.baidu.com/item/%E7%AB%8B%E4%BD%93%E5%A3%B0)**之分，单声道振幅数据为n\*1矩阵点，立体声为n\*2矩阵点，**取样频率**一般有11025Hz(11kHz) ，22050Hz(22kHz)和44100Hz(44kHz) 三种，不过尽管[音质](http://link.zhihu.com/?target=http%3A//baike.baidu.com/item/%E9%9F%B3%E8%B4%A8)出色，但在压缩后的文件体积过大！相对其他音频格式而言是一个缺点，其 **文件大小的计算方式**为：[WAV格式](http://link.zhihu.com/?target=http%3A//baike.baidu.com/item/WAV%E6%A0%BC%E5%BC%8F)文件所占**容量**（B) = （取样频率 X[量化位数](http://link.zhihu.com/?target=http%3A//baike.baidu.com/item/%E9%87%8F%E5%8C%96%E4%BD%8D%E6%95%B0)X 声道） X 时间 / 8 ([字节](http://link.zhihu.com/?target=http%3A//baike.baidu.com/item/%E5%AD%97%E8%8A%82)= 8bit) **每一分钟WAV格式的音频文件的大小为10MB，其大小不随音量大小及清晰度的变化而变化**。

注：专业名词（取样频率、量化位数、声道）解释：https://blog.csdn.net/eric88/article/details/17098603 

- **<font color="red">采样位数</font>：也叫量化位数(单位:比特)，**是存储每个采样值所用的二进制位数。采样值反应了声音的波动状态。采样位数决定了量化精度。**采样位数越长,量化的精度就越高,还原的波形曲线越真实,产生的量化噪声越小,回放的效果就越逼真**。常用的量化位数有4、8、12、16、24。量化位数与声卡的位数和编码有关。如果采样位数为16位，那么能够表示的范围就是$2^{16}$,采样位数越大，对声音的大小变化表现得也就越精细。
- **<font color="red">采样频率</font>：采样频率是指录音设备在一秒钟内对声音信号的采样次数，采样频率越高声音的还原就越真实越自然。**越高所能描述的**声波频率**就越高。**采样率**决定**声音频率**的范围（**相当于音调**），由采样定理得知采样率需为待采样声音频率的2倍。 
- **<font color="red">声道数</font>: 使用的声音通道的个数，**也是采样时所**产生的声音波形的个数**。播放声音时,单声道的WAV一般使用一个喇叭发声,立体声的WAV可以使两个喇叭发声。记录声音时,单声道,每次产生一个波形的数据,双声道,每次产生两个波形的数据,所占的存储空间增加一倍。
- **WAV格式大小：**采样率一般是44.1K，16bit采样精度，存储成**WAV格式大小** = 44.1KHz(采样率)  X 16bit(采样位数) X 2(双声道)  X  播放时间
- WAV格式是没有压缩无损的，MP3格式是按1:12压缩保存的，所以MP3格式大小等于上式的1/12。

<!--more-->

# WAV二进制格式解析

大部分的多媒体文件都依循着一种结构来存放信息，这种结构称为"资源互换文件格式"(Resources lnterchange File Format)，简称RIFF，例如声音的WAV文件，音频视频交错格式数据.AVI。

![img](https://img.singhe.art/817161-20190326145146250-1289168612.png)

以WAV格式为例，WAV文件一般由RIFF块、FMT块和DATA块三部分依次组成（还可能有JUNK块），下面分别进行介绍。

## RIFF块

```C++
typedef struct WAV_RIFF {
    char chunk_id_[4];     // "RIFF"
    uint32_t chunk_size_;  // 36 + sub_chunk2_size, 36是RIFF_T + FMT_T两个头信息的总大小
    char format_[4];       // "WAVE"
} RIFF_T;
```

- chunk_id_的值固定为"RIFF"，占用四个字节
- chunk_size_指示WAV数据的大小，包含RIFF块和FMT块的大小
- format_的值固定为"WAVE"，占用四个字节

## FMT块

```c++
typedef struct WAV_FMT {
    char sub_chunk1_id_[4];
    uint32_t sub_chunk2_size_;
    uint16_t audio_format_;
    uint16_t num_channels_;
    uint32_t sample_rate_;
    uint32_t byte_rate_;
    uint16_t block_align_;
    uint16_t bits_per_sample_;
} FMT_T;
```

- sub\_chunk1\_id_固定值为"fmt"
- sub\_chunk2\_size_代表`fmt` 块中后续格式数据的长度，后面剩下的字段(audio\_format\_等)加起来一共是16字节
- audio_format_：音频格式（1 = PCM）
- num\_channels\_：声道数（1 = mono，2 = stereo）
- sample_rate_：采样率，例如 44100
- byte_rate_： 每秒字节数 = sample\_rate\_ × num\_channels\_ × (bits\_per\_sample\_ / 8)
- block_align_：一个采样帧占用的字节数
- bits_per_sample_：每个采样的位数，例如 16 位

## DATA块

```c++
typedef struct WAV_DATA {
    char sub_chunk2_id_[4];
    uint32_t sub_chunk2_size_;
    // sub-chunk-data
    char data[0];
} DATA_T;
```

![img](https://img.singhe.art/817161-20190326184927550-508732460.png)

# C++程序解析格式

```c++
#include <iostream>
#include <cstdint>
#include <cstring>

typedef struct WAV_RIFF {
    char chunk_id_[4];     // "RIFF"
    uint32_t chunk_size_;  // 36 + sub_chunk2_size, 36是RIFF_T + FMT_T两个头信息的总大小
    char format_[4];       // "WAVE"
} RIFF_T;

typedef struct WAV_FMT {
    char sub_chunk1_id_[4];
    uint32_t sub_chunk2_size_;
    uint16_t audio_format_;
    uint16_t num_channels_;
    uint32_t sample_rate_;
    uint32_t byte_rate_;
    uint16_t block_align_;
    uint16_t bits_per_sample_;
} FMT_T;

typedef struct WAV_DATA {
    char sub_chunk2_id_[4];
    uint32_t sub_chunk2_size_;
    // sub-chunk-data
} DATA_T;

typedef struct WAV_FORMAT {
    RIFF_T riff_;
    WAV_FMT fmt_;
} WAV_HEADER;

void ShowWavHeaderInfo(const WAV_HEADER &header) {

}

int main() {
    FILE *fp = nullptr;
    const char *filepath = "/home/singheart/440hz.wav";
    fp = fopen(filepath, "rb");
    if (fp == nullptr) {
        fprintf(stderr, "open file [%s] failed\n", filepath);
        exit(-1);
    }
    WAV_HEADER header;
    fread(&header, sizeof(header), 1, fp);
    ShowWavHeaderInfo(header);

    return 0;
}
```

# Python画出WAV波形

```python
import wave as we
import numpy as np
import matplotlib.pyplot as plt

def wavread(path):
    wavfile =  we.open(path,"rb")
    params = wavfile.getparams()
    nchannels, sampwidth, framesra, frameswav = params[:4]
    print("声道数:%d" % nchannels)
    print("采样宽度（字节数）:%d" % sampwidth)
    print("采样率：:%d" % framesra)
    print("总帧数:%d（样本点数量）" % frameswav)
    datawav = wavfile.readframes(frameswav)
    wavfile.close()
    datause = np.frombuffer(datawav,dtype = np.short)
    if nchannels == 2:
        datause.shape = -1,2
    datause = datause.T
    # 总的样本点数量除以采样率（每秒采集多少个点）
    time = np.arange(0, frameswav) * (1.0/framesra)
    return datause, time, nchannels

if __name__ == "__main__":
    wavdata, wavtime, nchannels = wavread("/home/singheart/440hz.wav")

    duration = 0.01  # 只显示前0.01秒
    N = int(44100 * duration)
    plt.plot(wavtime[:N],wavdata[:N],color = 'green')
    plt.xlabel('Time')
    plt.ylabel('Amplitude')

    plt.show()
```
