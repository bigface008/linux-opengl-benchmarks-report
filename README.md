# Linux OpenGL Benchmark 总结报告

这个仓库简单记录了踩过的坑，以及一些测试脚本的总结。

## Steam

**在安装好、启动好Steam以及相应游戏之后**，通过以下命令启动游戏。

```bash
$ steam steam://rungameid/xxx
```

例如`steam steam://rungameid/550`可以启动求生之路2（Left 4 Dead 2）。游戏对应的gameid的可以通过`grep game_name ~/.local/share/Steam/SteamApps/*.acf`来查找。

**我建议一定要在Steam启动好、游戏装好后再执行这个命令！**

### 命令行启动，报错缺少lib

我有一次通过上面的命令启动不知为啥报错缺了这些lib。（明明之前在Steam图形化客户端里启动都没事）

```
libgtk-x11-2.0.so.0
libpulse.so.0
libgdk_pixbuf-2.0.so.0
libva.so.2
libbz2.so.1.0
libvdpau.so.1
libva.so.2
libva-x11.so.2
```

后来Steam重启的时候重启不了了，也说缺这些lib。最后只能卸载重装Steam和游戏。

完全卸载Steam请使用下面的命令。

```bash
$ sudo apt-get remove steam
$ sudo apt-get purge steam
$ rm -rf ~/.local/share/Steam && rm -rf ~/.steam
```

### 修改启动命令

在Steam图形化客户端中选择库，右键游戏，点击属性，设置启动选项。

### 游戏中文字体乱码

安装一些字体即可。

```bash
$ sudo apt-get install ttf-wqy-microhei ttf-wqy-zenhei xfonts-wqy
```

## 测量 FPS (Frame Rate)

目前测试过的工具包括

- [apitrace](https://github.com/apitrace/apitrace) 最有希望成功的。
- [libframetime](https://github.com/clbr/libframetime) 最简单的测试工具，折腾了很长时间后还是失败了。
- [glxosd](https://glxosd.nickguletskii.com/) 失败。不过后来觉得可能还有希望？
- [voglperf](https://github.com/ValveSoftware/voglperf)  Valve(Steam所属的公司)开发的开源FPS测试工具，仅能测试Steam上的游戏。停更了很长时间了。失败。

### apitrace

这个东西可以用`apitrace trace`命令来跟踪一个程序的图形库接口的调用，输出一个`.trace`文件，然后使用`apitrace dump`命令根据`.trace`文件来输出log。也可以通过`apitrace replay`来获取重播这个trace时的各个接口调用的cpu、gpu时间戳。

- [安装说明](https://github.com/apitrace/apitrace/blob/master/docs/INSTALL.markdown)
- [使用说明](https://github.com/apitrace/apitrace/blob/master/docs/USAGE.markdown)

对于一部分应用，apitrace需要经过一些调整才能使用。查看这个[链接](https://github.com/apitrace/apitrace/wiki/Applications)来了解。

如果要对Steam上的32位linux游戏进行trace，需要编译32位的apitrace。这个[链接](https://github.com/apitrace/apitrace/wiki/Steam)（一定要详细看！）里描述了相应的细节：除了编译32位版本，你还要

- 在wrappers中创建软链接。

  文档里说要`ln -s glxtrace.so wrapper/libGL.so*`，但是实际上glxtrace.so就是在wrapper文件夹里面的。我不知道这会有什么影响。

- 配置如下脚本

  ```bash
  #!/bin/sh
  export LD_LIBRARY_PATH="/path/to/apitrace/build32/wrappers:${LD_LIBRARY_PATH}"
  export TRACE_LIBGL="/usr/lib/i386-linux-gnu/libGL.so"
  /path/to/apitrace/build32/apitrace trace --api gl --output /path/to/output.trace "$@"
  ```

  假设脚本名称是`/path/to/run.sh`，你可以在游戏对应的启动命令里输入`/path/to/run.sh %command%`，接着在Steam客户端里直接启动游戏。

- 关闭Steam Community。

可以通过`file {可执行文件名}`这一命令来判断是32位还是64位，注意是二进制文件，不是shell脚本。

#### 编译32位的apitrace时遇到的问题

我直接按照安装教程去做，发现`make -C build32 glxtrace`这个命令会出错。[解决方法](https://github.com/apitrace/apitrace/issues/173)是

```bash
$ cd /usr/lib/i386-linux-gnu
$ sudo ln -s libX11.so.6 libX11.so
```

另外，需要运行`make -C build32`命令才能获得`build32/apitrace`可执行文件。教程里似乎没有提到这一点。

#### `apitrace dump`

trace记录完毕之后，

```bash
$ # 以求生之路2举例
$ /path/to/apitrace dump --calls=frame l4d2.trace > l4d2.frame.log
```

这样会输出所有和frame有关的调用，一般来说就是`glXSwapBuffer`，这样如果有时间戳就很容易计算FPS。

#### 给trace加上时间戳以计算FPS

需要注意的是，这东西在trace程序的时候，是不会记录各个接口调用的时间戳的！所以我们需要自己修改代码！

辛运的是，网上有人已经这么做过了。[apitrace原理分析及改进](https://blog.simbot.net/index.php/2018/01/28/apitrace2/)这篇文章的末尾提到了修改apitrace来使它能够在trace时就能记录时间戳，提供了相应的`diff`文件:

- [cputime-diff](https://blog.simbot.net/wp-content/uploads/2018/01/cputime-diff.txt)
- [gui-diff](https://blog.simbot.net/wp-content/uploads/2018/01/gui-diff.txt)
- [other-diff](https://blog.simbot.net/wp-content/uploads/2018/01/other-diff.txt)

你可以通过`git apply`命令来直接更新apitrace的代码。需要注意的是他列出的`other-diff`文件中使用的是Python2的代码，而最新的apitrace需要Python3。所以我们要自己加上`print`语句的括号。

我照着这篇文章改了之后，发现会在执行`apitrace dump`时报错

```
error: (glXGetProcAddressARB) unknown call detail 6
```

目前还不知道怎么解决。

总之，目前可以成功在glxgears、glmark2、求生之路2（Left 4 Dead 2）上运行trace，并且dump，但是改了代码之后求生之路2就不能dump 。

## CPU & Memory

每秒输出一次pid、cpu占用率、内存占用率、时间戳、运行benchmark的命令。

```bash
#!/bin/sh
# 1st arg: pid / name of benchmark (It's recommeded to use the pid).
# 2nd arg: file name used for output.
while [ true ]; do
/bin/sleep 1
# ps -ux | grep $1 | grep -v $0 | grep -v 'grep' >> test.txt
ps -eo pid,%cpu,%mem,etime,command | grep $1 | grep -v $0 | grep -v 'grep' >> $2
done
```

使用例：

```bash
$ ./log_cpu_mem.sh {pid} cpu_mem.log
```

## GPU

这个脚本只能测总的gpu使用率和显存占用，不能指定具体进程。要用的话，可以在运行benchmark前用`nvidia-smi`测试一下目前的显存占用。

```bash
#!/bin/sh
nvidia-smi --query-gpu="timestamp,utilization.gpu,memory.used,memory.total" -l 1 -f $1 --format=csv
```

使用例：

```bash
$ ./log_gpu.sh gpu.log
```

不过`nvidia-smi`这个工具也有办法每秒更新（输出）一次显存占用的方法，但是那个输出格式不是一行一行的。（这里就不讲了，具体请看文档`man nvidia-smi`。）
