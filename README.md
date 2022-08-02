# mse-demo

Simple working example using the Media Source Extensions (MSE) to playback video

mse-demo是fork自`bitmovin/mse-demo`，感谢原作者。



# 4k播放的本地dash服务


## 生成fragment mp4



### 下载安装bento4



Bento4是一个C++类库和工具，用于读取和写入ISO-MP4文件。除了支持ISO-MP4外，Bento4还支持解析和复用H.264和H.265 ES流，将ISO-MP4转换为MPEG2-TS，打包HLS和MPEG-DASH、CMAF、内容加密、解密等。



```bash
# http://zebulon.bok.net/Bento4/binaries/

wget http://zebulon.bok.net/Bento4/binaries/Bento4-SDK-1-6-0-639.x86_64-unknown-linux.zip
unzip Bento4-SDK-1-6-0-639.x86_64-unknown-linux.zip

cd Bento4-SDK-1-6-0-639.x86_64-unknown-linux/

# 直接放到/usr目录下
sudo cp * /usr/. -rf
```



输入mp4，tab键补全，可以看到，可用的binary都能找到了：

```bash
$ mp4
mp42aac         mp42hevc        mp42ts          mp4dash         mp4dcfpackager
mp4dump         mp4encrypt      mp4fragment     mp4iframeindex  mp4mux          
mp4split        mp42avc         mp42hls         mp4compact      mp4dashclone 
mp4decrypt      mp4edit         mp4extract      mp4hls          mp4info 
mp4rtphintinfo  mp4tag    
```



### 转换mp4为fragment mp4



使用`mp4fragment`，将普通mp4转换为fragment mp4，不指定`--fragment-duration`，生成的fragment duration按default值:

```bash
$ mp4fragment 4k-no_audio.mp4  fragmented-default.mp4 
found regular I-frame interval: 1511 frames (at 29.970 frames per second)
auto-detected fragment duration too large, using default
```



指定fragment时长为2s：

```
mp4fragment --fragment-duration 2000 4k-no_audio.mp4 fragmented.mp4
```



通过mp4info查看生成的fragmented.mp4文件信息，这时候fragments字段已经变成yes：

```bash
$ mp4info fragmented.mp4 
File:
  major brand:      isom
  minor version:    200
  compatible brand: isom
  compatible brand: iso2
  compatible brand: avc1
  compatible brand: mp41
  compatible brand: iso5
  fast start:       yes

Movie:
  duration:   97903 (media timescale units)
  duration:   97903 (ms)
  time scale: 1000
  fragments:  yes
```



### fragment mp4分片



对fragment mp4进行分片:

```bash
$ mp4dash fragmented.mp4 
Parsing media file 1: fragmented.mp4
WARNING: video segment durations for "File 1#1" vary by more than 10% (consider using --use-segment-timeline)
Splitting media file (video) fragmented.mp4
```



分片完成后，没有指定output-dir参数，会生成output目录：

```bash
$ cd output/video/avc1
$ ls
init.mp4    seg-11.m4s  seg-13.m4s  seg-15.m4s  seg-17.m4s  seg-19.m4s 
seg-20.m4s  seg-22.m4s  seg-3.m4s  seg-5.m4s  seg-7.m4s  seg-9.m4s
seg-10.m4s  seg-12.m4s  seg-14.m4s  seg-16.m4s  seg-18.m4s  seg-1.m4s
seg-21.m4s  seg-2.m4s   seg-4.m4s  seg-6.m4s  seg-8.m4s
```



## 配置mse-demo

### 4k.html文件

新建4k.html文件，修改baseUrl，templateUrl字段，放到nginx服务的目录下：



```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>MSE Demo</title>
</head>
<body>
  <h1>MSE Demo</h1>
  <div>
    <video controls width="80%"></video>
  </div>

  <script type="text/javascript">
    (function() {
      var baseUrl = 'http://192.168.31.124/dash/output/video/avc1/';
      var initUrl = baseUrl + 'init.mp4';
      var templateUrl = baseUrl + 'seg-$Number$.m4s';
      var sourceBuffer;
      var index = 1;
      var numberOfChunks = 22;
      var video = document.querySelector('video');

      if (!window.MediaSource) {
        console.error('No Media Source API available');
        return;
      }

      var ms = new MediaSource();
      video.src = window.URL.createObjectURL(ms);
      ms.addEventListener('sourceopen', onMediaSourceOpen);

      function onMediaSourceOpen() {
        sourceBuffer = ms.addSourceBuffer('video/mp4; codecs="avc1.4d401f"');
        sourceBuffer.addEventListener('updateend', nextSegment);

        GET(initUrl, appendToBuffer);

        video.play();
      }

      function nextSegment() {
        var url = templateUrl.replace('$Number$', index);
        GET(url, appendToBuffer);
        index++;
        if (index > numberOfChunks) {
          sourceBuffer.removeEventListener('updateend', nextSegment);
        }
      }

      function appendToBuffer(videoChunk) {
        if (videoChunk) {
          sourceBuffer.appendBuffer(new Uint8Array(videoChunk));
        }
      }

      function GET(url, callback) {
        var xhr = new XMLHttpRequest();
        xhr.open('GET', url);
        xhr.responseType = 'arraybuffer';

        xhr.onload = function(e) {
          if (xhr.status != 200) {
            console.warn('Unexpected status code ' + xhr.status + ' for ' + url);
            return false;
          }
          callback(xhr.response);
        };

        xhr.send();
      }
    })();
  </script>
</body>
</html>

```



### 播放测试

```bash
$ google-chrome http://192.168.31.124/mse-demo/4k.html
```

