---
title: live_media_audio
date: 2021-02-18 02:28:34
tags: audio
categories: audio
---


### 音频科技发展史：
#### 声音的本质：
声音的本质是一种振动，可以说是一种一维的物理信号；可以通过敲击，让物体振动产生，并且可以使空气产生振动，从而传播，不过会能量衰减，导致只能传播一定的距离，原始振动
能量越大，传输的距离就越大；<!--more-->
声音作为这种波动类型的物理信号的特点和表示：振幅，频率，相位，波长，共振等等，这些不赘述，有兴趣再翻阅资料；
#### 音频的录制发展史：
*豆瓣有篇文章说的还可以：https://www.douban.com/group/topic/28212958/*
+ 简单来说，声音的录制经历了一下的时代:  
声音-<->动能(机械)的时代： 即通过声波振记器，比如：可以将声波变换成金属针的震动，然后将波形刻录在圆筒形腊管的锡箔上。当针再一次沿着刻录的轨迹行进时，便可以重新发出留下的声音。  
声音<-->电信号的时代：即：声音-->振动--麦克风振动膜放大-->线圈磁铁-->电信号磁信号： 由磁带记录，播放；  
声音<-->数字信号和处理时代：即声音-->麦克风-->电信号-->模拟信号处理AD/DA转换成数字信号：具体学习数字信号和模拟信号处理，了解示波器等器件；-->101010的数字信号-->  
电脑接收和处理-->数字信号转模拟信号AD/DA -->转为声音信号-->扬声器  增加了立体声；
PS:https://zhuanlan.zhihu.com/p/64050348  
ADC=Analog Digital Change 模数转换
DAC= Digital Analog Change 数模转换
#### 现代声音录制采集处理的基本过程：
即上述的第三个时代，这里稍微详细解释：现代为了提高声音的质量，在采集上也有各种细分： 环境的保证，声音无损传递，放大，麦克风的设计和材料，声音芯片的设计和材料(语音芯片有多种，涉及
声音从模拟信号到数字信号的过程：AD/DA转换), 采样也有相关定理：奈奎斯特采样定律；(模拟信号数字化必须经过三个过程，即抽样、量化和编码，以实现话音数字化的脉冲编码调制（PCM，Pulse Coding Modulation）技术。)，至此，完成声音的数字化过程，但是对声音的处理还远不止于此，采集到的声音可能有噪声，对噪声的处理，降噪技术等，采集到的声音的转换，混合，传输
识别，智能转文本等等。


### 音频的组成：
#### 音频的表示和基本参数：
          + 简单介绍PCM 声音如何从模拟信号转换为数字信号：  
{% asset_img pcmall.png This is an example image %}
          + 采样：采样是将模拟信号以其带宽两倍以上的频率提取样值，变为在时间轴上离散数据的过程：
             采样率：每秒从连续信号中提取出并组成离散信号的采样个数：用Hz表示；  
             如：
		如音频信号采样率为8000hz。
		可以理解上图采样对应图中 那段电压随时间变化的曲线 为1秒 那下面那个1 2 3 …10那就因该有1-8000个点，即将1秒均分为8000份，依次取出来那8000个点时间 对应的电压值。
          + 量化：
             可以看到，在时间轴上连续的值已经变为离散的了，但是在电压上(y轴)上的值还是可能有无限多个值的情况，所以这个时候需要将电压上的值进行量化,举个例子：
             采样位数： 即描述数字信号所用的位数：如t1时间的电压值V1用8bit的数值表示： 3： 00000011 ，8bit最大可以表示数值为256,16位类似；
             量化精度，即将一个范围的值转换为另一个范围的值，两个值之间的间隔： 比如：将0-3.3V的范围电压值存储在8位的数字里即： 3.3/256=0.0128
             量化： 即比如将0-3.3V的电压值放到8位数字中：即 电压值1.65V对应的值就是128，以此类推；
             量化的后果：量化后的抽样信号与量化前的抽样信号相比较，当然有所失真，且不再是模拟信号。这种量化失真在接收端还原模拟信号时表现为噪声，并称为量化噪声。量化噪声的大小取决于把样值分级“取整”的方式，分的级数越多，即          量化级差或间隔越小，量化噪声也越小。

          + 编码：将量化后得到的类似十进制数字码流经过一定的规则转换为二进制码流进入数字系统的过程；  
             常见的有PCM音频编码：
             PCM协议：  
	PCM（PCM-clock、PCM-sync、PCM-in、PCM-out）脉冲编码调制，模拟语音信号经过采样量化以及一定数据排列就是PCM了。理论上可以传   输单声道，双声道立体声和多声道。是数字音频的raw data.
             PCM信号：PCM信号未经过任何编码和压缩处理(无损压缩)。与模拟信号比，它不易受传送系统的杂波及失真的影响。动态范围宽，可得到音质相当好的效果。编码上采用A律13折线编码。
            关于双声道的采样：  
{% asset_img channel.png This is an example image %}
            +  关于采样频率：
             人对频率的识别范围是 20HZ - 20000HZ, 如果每秒钟能对声音做 20000 个采样, 回放时就足可以满足人耳的需求.
                8000hz 为电话采样。
                22050 的采样频率是常用的。
                44100已是CD音质, 超过48000的采样对人耳已经没有意义
                对采样率为44.1kHz的AAC（Advanced Audio Coding）音频进行解码时，一帧的解码时间须控制在23.22毫秒内。通常是按1024个采样点一帧。
                而一个采样点，可以理解为就是一个8bit/16bit的音频值/hz
                PS： 音频其实没有帧的概念，音频是一连串的采样值，比如采样率为44.1kHZ，采样精度为16位的音频，你可以算出bitrate（比特率）是4410016kbps，每秒的音频数据是固定的4410016/8 字节。而一般是每次返回1024个采样值；
          + 什么样的采样和音频值位数叫无损呢？
##### 音频的采集和表示：
+ pcm的格式的raw data数据；
          在linux下的采集
          1）采集白噪音，播放并导入到/null  cat /dev/urandom | padsp tee /dev/audio > /dev/null 
          2）采集内存内容到播放audio,和/null cat /dev/mem | padsp tee /dev/audio > /dev/null 
          3)  采集白噪音并播放和保存：cat /dev/urandom | padsp tee /dev/audio > /home/some.raw
          注意这里是按默认的采样率，声道数，量化位数的；
          更多接口：音频的linux下的编程接口；https://docs.google.com/viewerng/viewer?url=https://www.programmer-books.com/wp-content/uploads/2019/06/Linux-Sound-Programming.pdf
+ 音频音量，pcm格式等表示；见上
+ 音频编码情况：
WAV:pcm是无损wav文件中音频数据的一种编码方式，但wav还可以用其它方式编码
##### 音频的基本参数：采样率，音频值位数，比特率等；
#### 音频查看的相关工具：Audacity 可以导入原始格式进行播放和查看：
将wav或其他格式转换为pcm: eg:
ffmpeg -i file.wav -f s16be -ar 8000 -acodec pcm_s16be file.raw
PCM数据格式：即原始的音频采样数据：
比如：在得到file.raw后，用audacity播放，用文本编辑器打开，删除一半或一些数据，保存后再次用audacity播放，即可得到剩一半的数据；
WAV数据格式：PCM的基础上加上一些头，meta信息
其他工具：praat


#### 认识音频原始数据
##### 在windows/linux上采集pcm原始音频
1） 列出设备：
```
   ffmpeg -list_devices true -f dshow -i dummy  
   在DirectShow audio devices下  
   eg: "xxx"(Realtek Audio)" 中文时可能有乱码，这个时候可以选择可选的另一个名字：  
   eg: Alternative name "@device_cm_{33D9A762-90C8-11D0-BD43-00A0C911CE86}\wave_{E41DED46-6012-4493-8D55-7D9322A433B6}"
```

2）通过展示出来的设备进行录制音频：  
```
ffmpeg.exe -f dshow -t 20 -i audio="@device_cm_{33D9A762-90C8-11D0-BD43-00A0C911CE86}\wave_{E41DED46-6012-4493-8D55-7D9322A433B6}"  -f s16le -y D:\sijiruni3.pcm 
```


3) 按格式播放采集的音频：  
```
ffplay.exe -channels 2 -f s16le -i D:\sijiruni.pcm
将pcm转为wav:ffmpeg.exe -f s16le -ar 44100 -ac 2 -i D:\audiotest\output_halfleft.pcm out_halfleft.wav
```
注意：若播放失败，尝试用大端/小端都尝试下；

##### 参数解释：
```
 -f  s16le  : 
    f: format 
    s16  ： signed 16bit
    u16: unsigned 16bit 
    le : little end  小端
    be: big end 大端
```
更多格式：
```
$ ffmpeg -formats | grep PCM
 DE alaw            PCM A-law
 DE f32be           PCM 32-bit floating-point big-endian
 DE f32le           PCM 32-bit floating-point little-endian
 DE f64be           PCM 64-bit floating-point big-endian
 DE f64le           PCM 64-bit floating-point little-endian
 DE mulaw           PCM mu-law
 DE s16be           PCM signed 16-bit big-endian
 DE s16le           PCM signed 16-bit little-endian
 DE s24be           PCM signed 24-bit big-endian
 DE s24le           PCM signed 24-bit little-endian
 DE s32be           PCM signed 32-bit big-endian
 DE s32le           PCM signed 32-bit little-endian
 DE s8              PCM signed 8-bit
 DE u16be           PCM unsigned 16-bit big-endian
 DE u16le           PCM unsigned 16-bit little-endian
 DE u24be           PCM unsigned 24-bit big-endian
 DE u24le           PCM unsigned 24-bit little-endian
 DE u32be           PCM unsigned 32-bit big-endian
 DE u32le           PCM unsigned 32-bit little-endian
 DE u8              PCM unsigned 8-bit
 https://trac.ffmpeg.org/wiki/audio%20types
```

##### 音频采集常见参数解释：
```
 channel: 通道数，可以有多个通道支持，常见有，单声道和立体声(两个通道)
rate： 采样率： 即每秒采样的点数，比如44100hz,则每秒采集44100个值，这个值大小代表振幅大小，即音频音量值
位深： 即一个采样点用多大类型的值表示： 常见有8bit,16bit等，见上；
frame: 在单声道：一个帧表示一个通道下的一个采样，而双通道，则一个帧包含两个采样点；所以帧率和sample rate采样率一样；
           比如：单声道： 44100hz,则表示 ，每秒44100个采样点，帧率为44100/s,而 双通道下，44100hz,由于一帧两个采样，所以帧率也是44100；
           注意这里的帧率和传输上的帧率不同；
period time: 毫秒单位，两个硬件中断的时间间隔，这个中断用来刷新缓存；
period size: 硬件中断之间的帧数量；和以下相关：
          period time = period size * time/frame  : 帧数*每帧占用的时间长度；
                             = period size * number of channels * time/sample : 帧数* 每个采样占用的时长
                            = period size * number of channels / samplate rate : 帧数/采样率(帧率)
            eg:  48000hz 双通道，periodsize = 8192frames,则 period time = 8192/48000=170.5ms
periods: 每个buffer的periods数量
buffer time: 一个buffer的时长
buffer size: 一个buffer的帧数；
    一个buffer的时长= buffer size * time/frame ： 一个buffer的帧数* 一个帧的时长
                              = buffer size * number of channels * time/sample : 一个buffer帧数 * channel数* 一个采样点的时长
                              = buffer size * number of channels / sample rate

注意一个buffer可能是多个periods;
https://www.alsa-project.org/wiki/FramesPeriods  更多更详细解释
```


##### 音频数据组织方式和解释：
音频中的数据如何排放：  
对单声道： 每个采样点一个数据，以16bit，44100hz为例，XXOO  OOXX .... ,1s有44100个值  
对双声道： 每个采样点一个数据，PCM16LE双声道数据中左声道和右声道的采样值是间隔存储的.  
注意： 对单声道来讲，也是两个耳机都能播放的，关键在于左右声道平衡的设置，如audacity  
而当双声道下，一个声道的数据为0时，则会出现左/右 只有一个有声音的情况；默认播放时 ；  
所以我们可以将音频数据左右交替清除来达到空间的效果；  
```cpp
eg:
只有右耳能听到；
int simplest_pcm16le_halfvolumeleft(char *url){
	FILE *fp=fopen(url,"rb+");
	FILE *fp1=fopen("output_halfleft.pcm","wb+");
 
	int cnt=0;
 
	unsigned char *sample=(unsigned char *)malloc(4);
 
	while(!feof(fp)){
		short *samplenum=NULL;
		fread(sample,1,4,fp);
 
		samplenum=(short *)sample;
		*samplenum=0;
		//L
		fwrite(sample,1,2,fp1);
		//R
		fwrite(sample+2,1,2,fp1);
 
		cnt++;
	}
	printf("Sample Cnt:%d\n",cnt);
 
	free(sample);
	fclose(fp);
	fclose(fp1);
	return 0;
}

https://blog.csdn.net/leixiaohua1020/article/details/50534316
```

##### 音频技术：
+ 混音  TODO
+ 重采样 TODO

##### 音频压缩：
TODO

##### alsa音频编程：
ref linux-sound-programming.pdf
mark:
```
对alsa编程编译运行成功，但是貌似录制不到声音，用了arecord也不行；
sudo arecord -D "hw:0,0" -f S16_LE -r 16000 -c 2 -d 20 -t wav test2.wav
./test hw:0.0 file.pcm
也都不行，但是其实命令和设备都是对的；
先不搞了，本来还想依赖它搞硬件的混音，这里不了，直接录制后，从pcm或其他级别来搞混音；
```

#### next:
```
1）学习audacity和理解各个音频功能： 混音，音频重采样等
2）学习音频相关协议： aac,wav,ogg,flac等；
3)  学习如何编程处理，裸，或者用ffmpeg库；
```
