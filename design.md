# Monibuca设计原理

## 背景

市面上的流媒体服务器不可谓不多，从本人的第一份工作起，就一直接触和研究了形形色色的流媒体服务器，从最早的**FCS**(全称Flash Communication Server),后来改名为**FMS**(全称Flash Media Server),到Red5(java语言开发)，到**CrtmpServer**(C++开发)，让我对流媒体服务器的基本原理有了深刻的认识。当时本人痴迷C#，于是乎在业余时间对crtmpServer的代码进行移植，用C#仿照着写了一遍取名为[csharprtmp](https://github.com/langhuihui/csharprtmp)，并且适当的增强了一些功能，于是对rtmp协议了如指掌。后来Adobe推出了RTMFP协议，是一种p2p协议，十分节省带宽。我就又开始研究一款名为**OpenRTMFP**的开源项目，后来该项目改名为**MonaServer**。我在起基础上进行了扩展，实现了一些例如录制flv，shareObject等原本FMS有的功能。后开发出了HTML5直播技术（现在命名为Jessibuca,尚未开源)，采用的传输协议就是WebSocket传输裸的视频流的方式，属于私有协议。而Server当时就使用的MonaServer。但当时遇到一个问题，C++的内存泄漏问题，这个一直没有很好的解决。遂决定放弃使用MonaServer转而使用srs，而srs要用一个很简单的go写的小程序将http-flv转换成WebSocket的Flv来适配我的Jessibuca，感觉最好能直接修改srs来实现这个功能。对srs的源码研究了一小段时间后放弃了，因为C++代码过于难写，容易出现bug。后来转而使用golang写的**gortmp**作为server，同样对其进行了扩展，而且进展十分顺利，golang的开发效率令人惊叹，而且其协程的特性很完美的处理了流媒体服务器的并发的场景。所以使用golang写的流媒体服务器项目很多，github上随便一搜就有很多，比如**livego**、**joy4**等。期间还接触到一位使用Node.js实现的流媒体服务器Node Media Server，我也和作者交流了许多，收益良多。

### 现有项目的不足
虽然流媒体服务器项目很多，但在我使用过程中遇到了几个痛点

1. 功能太多太重，往往大而全，不够轻量 很多号称轻量的项目最后都会越来越重
2. 扩展性弱，由于功能复杂，设计之初没有提供良好的扩展性，有些项目带有脚本支持，如FMS和MonaServer，但执行脚本会牺牲性能，而且脚本和原生代码相比，功能限制很大，只能实现业务逻辑而不是流媒体服务器本身的功能扩展。
3. 缺少图形管理界面，FMS是配套有图形管理界面的，当然FMS的问题是商业软件需要付费，源码也是不可见的。

综上所述，本人在吸收了以上诸多流媒体服务器的设计后，完成了Monibuca这款golang编写的流媒体开发框架的编写

### 受到vue渐进式思想的影响
vue渐进式框架的设计思想非常棒，那么是否可以用来设计流媒体服务器，使得流媒体服务器不只是一个服务器，而是一个开发框架，让开发者可以定制化自己的流媒体服务器呢？答案是肯定的。当然我们需要更多的抽象。

## 如何实现可扩展——插件化
许多IDE和编辑器都依靠插件化技术得以拓展其功能，并形成其生态，例如vs、vs code、eclipse、jetbrains系列，当然vue作为一个前端框架也是设计了很不错的插件机制。这些都可以作为借鉴。

要实现流媒体服务器的插件化，就需要把核心功能和拓展功能分离，进行足够的抽象。

### 三大抽象概念
1. 发布者（Publisher）
2. 订阅者（Subscriber）
3. 房间（Room）

#### 发布者（Publisher）
发布者本质上就是输入流，其抽象行为就是将音频和视频数据压入**房间**中，换句话说，就是在恰当的时候调用**房间**的PushVideo和PushAudio函数
::: tip 源码位置
发布者定义位于monica/publisher.go中
:::
在发布者的定义中有一个**InputStream**的结构体，用来和**房间**进行互操作。
所有具体的发布者都应该包含这个**InputStream**，以组合继承的方式成为发布者。
该**InputStream**包含最核心功能就是Publish函数，这个函数的功能就是在**房间**里面设置发布者是自己，这个行为就是发布。形象的理解就是主播走进了房间。
引擎不关心是谁走进了房间，也不关心进来的人会发布什么内容。
::: tip 发布者插件
所有实现了发布者具体功能的插件，就是发布者插件，这样一来，流媒体的媒体源可以是任意的形式，比如RTMP协议提供的推流，可以由FFMPEG、OBS发布。也可以是读取本地磁盘上的媒体文件，也可以来自源服务器的私有协议传输的内容。
:::
#### 订阅者（Subscriber）
订阅者就是输出流，其抽象行为就是被动接收来自**房间**的音频和视频数据。
::: tip 源码位置
订阅者定义位于monica/subscriber.go中
:::
订阅者有两个函数sendVideo和sendAudio用于接收音频和视频数据。这个两个函数会对音视频做一些预处理，主要是实现丢包机制、时间戳和首屏渲染。具体的视频数据会共享读取。
然后调用SendHandler将打包好的音视频数据发送到具体的订阅者那里。
::: tip 订阅者插件
订阅者插件，本质上就是SendHandler函数。具体可以将打包的数据以何种协议输出，还是写入文件，由插件实现。
:::

#### 房间（Room）
房间就是一个连接发布者和订阅者的地方。可以形象的理解为主播的房间，发布者是主播，订阅者就是粉丝观众。房间是引擎的核心，其重要逻辑包括：
1. 房间的创建、查询、关闭
2. 订阅者的加入和移除
3. 发布者的进入和离开。
::: tip 源码位置
订阅者定义位于monica/room.go中
:::

流媒体服务器的核心是**转发**二字。当你去研究一款流媒体服务器的时候，会有海量的代码阻碍你看清其核心逻辑。包括：

1. 多媒体格式定义、解析，如Flv、MP4、MP3、H264、AAC等等
2. 传输协议的解析，如RTMP家族、AMF、HTTP、RTSP、HLS、WebSocket等等
3. 各种工具类，用来读取字节的缓冲、大小端转换、加解密算法、等等

大部分流媒体服务器都是基于rtmp协议之上扩展而来，这是历史原因造成的，所以功能不能很好的分离，耦合度很高。往往牵一发而动全身。其实所谓的流媒体服务器本质上就是把发布者的数据经过服务器转发到订阅者手里播放，起一个中转作用。至于什么协议格式，什么媒体格式都是属于扩展功能。所以最轻量的服务器应该不包含任何协议格式，任何媒体格式，仅仅只是完成中转。再说的直白一点核心代码就是一个for循环。
```go
for _, v := range r.Subscribers {
    v.sendVideo(video)
}
```
其他都是围绕这个for循环展开。所有的流媒体服务器代码里面都有这个for循环，写法稍有不同，但本质相同。
::: tip 源码位置
该核心逻辑位于monica/room.go中的Run函数内
:::

## 如何实现高性能
流媒体服务器对性能要求极为苛刻。因为流媒体服务器属于高速系统，会有并发的长连接请求，协议封包解包和音视频格式的编解码都消耗着CPU以及内存，如何尽可能的减少消耗是必须考虑的问题。

### 内存使用
池化是一个不错的选择，所以尽量池化，在Monibuca中对`[]byte`类型，采用了[github.com/funny/slab](https://github.com/funny/slab)包来管理。其他结构体就用系统自带的pool包来池化对象。

### 协程的使用
golang自带的goroutine可以有效的减少线程的使用，并可以支持各种异步并发的情况。合理的创建goroutine很重要，这样才能尽可能高效利用CPU时间。
在monibuca中，创建goroutine在如下场景中：
1. 通讯协议建立的长连接对于一个goroutine
2. 每个房间拥有一个goroutine用于接收指令和转发音视频数据
3. 每一个插件会使用一个goroutine来执行插件的Run函数

由于引擎本身比较轻量化，更多的性能的优化需要插件提供者自由发挥了。