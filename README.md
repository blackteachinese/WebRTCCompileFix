# webrtc.org源码如何编译iOS使用的Framework
## 编译工具准备
* depot_tools
    * depot_tools是Chromium 和 Chromium OS管理代码拉取的脚本包。里面包含了gclient, gcl, git-cl, repo
* 创建一个build工作目录
    * /webrtc_build
* 在webrtc_build目录下，使用git下载Chromium的depot_tools工具

```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git

```
* 添加depot_tools到你OS的环境变量

```
export PATH=`pwd`/depot_tools:"$PATH"
```


## 获取源码
在webrtc_build目录下，使用命令行拉取webRTC的源码

```
fetch --nohooks webrtc_ios
gclient sync
```

得到源码：src/...

## 开一个新的分支

```
git new-branch <branch name>
```

## 使用GN生成Ninja工程文件

```
# debug build for 64-bit iOS
gn gen out/ios_64 --args='target_os="ios" target_cpu="arm64"'
# debug build for simulator
gn gen out/ios_sim --args='target_os="ios" target_cpu="x64"'
```

## 命令行编译Ninja工程

```
ninja -C out/ios_64 AppRTCMobile
```

AppRTCMobile是其中一个Target,可以切换别的

## 生成Xcode工程

```
gn gen out/ios --args='target_os="ios" target_cpu="arm64"' --ide=xcode
open -a Xcode.app out/ios/all.xcworkspace
```

## 编译webRTC.framwork
* 查看编译选项

```
./src/tools-webrtc/ios/build_ios_libs.py --help
```

* 默认会编译出全部架构的 ['arm64', 'arm', 'x64', 'x86'] 的 framework.

```
./src/tools-webrtc/ios/build_ios_libs.py
```

* 只编译ARM架构


```
./src/tools-webrtc/ios/build_ios_libs.py --arch {'arm64','arm'}
```


![](http://ww2.sinaimg.cn/large/006tNc79gy1feses56wbmj31gs0j6q8t.jpg)

* xcode8.3的编译错误

如果你的XCode版本是8.3，执行build_ios_libs.py会出现下面的错误

~~~
 error: taking address of packed member 'time_entered' of class or structure 'sctp_state_cookie' may result in an unaligned pointer value [-Werror,-Waddress-of-packed-member]
~~~

![](http://ww3.sinaimg.cn/large/006tNc79gy1fesetelf94j31hg0twdq7.jpg)

原因是有两个文件有错需要改正：
third_party/usrsctp/usrsctplib/usrsctplib/netinet/sctp_input.c
third_party/usrsctp/usrsctplib/usrsctplib/netinet/sctp_output.c
具体改正点下面可以查看
[sctp_input.c改正](https://trac.webkit.org/changeset/211848/webkit/trunk/Source/ThirdParty/libwebrtc/Source/third_party/usrsctp/usrsctplib/usrsctplib/netinet/sctp_output.c)
[sctp_output.c改正](https://trac.webkit.org/changeset/211848/webkit/trunk/Source/ThirdParty/libwebrtc/Source/third_party/usrsctp/usrsctplib/usrsctplib/netinet/sctp_input.c)

* 编译配置文件错误
  你把编译好的WebRTC.framework放到Demo工程跃跃欲试，却可能发现编译不过。
  日了🐶了。。。
   "Could not build module 'WebRTC'"
   "'WebRTC/RTCVideoCapturer.h' file not found"
  ![](http://ww3.sinaimg.cn/large/006tNc79gy1fesetv8i40j31jm0t8qhw.jpg)
  
  太可怕了。。看来编译的脚本又有问题，欲哭无泪。。。
  一通找终于找到生成WebRTC.framework头文件的脚本。
  ![](http://ww3.sinaimg.cn/large/006tNc79gy1fesesq52cmj30tu18mgwa.jpg)
  
  缺了下面几个头文件链接
  
```
"objc/Framework/Headers/WebRTC/RTCCameraVideoCapturer.h",
"objc/Framework/Headers/WebRTC/RTCNSGLVideoView.h",
"objc/Framework/Headers/WebRTC/RTCVideoCapturer.h",
```

修改BUILD.gn，再使用build_ios_libs.py编译，就可以得到完美的WebRTC.framework。

* 把编译好的WebRTC.framework导入demo工程，编译demo工程,发现动态库找不到。

```
dyld: Library not loaded: @rpath/WebRTC.framework/WebRTC
```

在Embedded Binaries +WebRTC.framework即可

最后附上修改后的文件BUILD.gn、sctp_input.c、sctp_output.c
以及编译成功后的webRTC.Framework(arm和All)

[git地址：https://github.com/blackteachinese/WebRTCCompileFix](https://github.com/blackteachinese/WebRTCCompileFix)





