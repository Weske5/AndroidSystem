# 小肩膀教育安卓定制系统
> 持续输出高质量课程，课程咨询小肩膀QQ：24358757  
> 小肩膀定制系统百度网盘地址：
> 链接：https://pan.baidu.com/s/1bfkY8MOKdWGpVRWa522C2A 提取码：9822

## 定制系统目前的功能：
1. 防root检测
2. 支持直接替换修改系统文件
3. 一定程度防vpn检测
4. 一定程度防aosp检测
5. WebView强制可调试
6. Frida持久化
7. 脱整体加固壳
8. 脱抽取加固壳
9. 追踪函数调用
10. smali trace

## 使用方法：
1. 目前支持两个设备
    pixel 1(sailfish)
    pixel 3(blueline)
2.  刷机前的准备
    - 2.1  解锁bootloader锁
    - 2.2  通过USB连接手机至电脑
        需要打开USB调试，找不到设备的，还需要安装谷歌USB驱动，驱动可通过Android Studio下载。
    - 2.3 配置好adb和fastboot
        将Android SDK里的platform-tools路径，添加到环境变量PATH中。
3.  运行刷机脚本即可
    - Windows运行flash-all.bat
    - Linux和Mac运行flash-all.sh
4. 刷机包说明  
   - blueline刷机包里没有去掉su，可以先将system挂载为可读写，然后给su改个名字即可，定制系统里的adb本身就是root权限，不需要su。

## 功能说明：
### 1. 抓包证书移动到系统目录
   - 1.1 将证书安装到用户目录    
从Charles中保存根证书，这里将其保存为Charles.pem  
将证书推送到手机sdcard目录
`adb push Charles.pem /sdcard/`  
打开设置 --> 安全 --> 加密与凭据 --> 从存储设备安装  
选择sdcard目录，然后点击Charles.pem证书安装
   - 1.2 移动证书到系统目录  
     依次运行以下adb命令，挂载system为可读写
```shell
    adb root
    adb disable-verity
    adb reboot
    adb remount
```  

   移动用户目录证书到系统证书目录  
   `mv -f /data/misc/user/0/cacerts-added/* /system/etc/security/cacerts`  
   删除用户凭据  
   打开设置 --> 安全 --> 加密与凭据 --> 用户凭据 --> 选择一个用户凭据删除
   - 1.3 证书效期  
   注意：证书是有效期的，Charles证书一般是一年，过了有效期，按上述方法重新操作  
   证书效期可按以下方法查看：  
   打开设置 --> 安全 --> 加密与凭据 --> 信任的凭据 --> 系统 --> 点击证书

### 2. Frida持久化
   将GadgetConfig.js推送至 `/data/local/tmp/xiaojianbang/`包名，即开启该功能  
   注意事项  
   GadgetConfig.js文件名称不可更改。  
   GadgetConfig.js中的内容与Frida官网介绍的配置内容一致。  
   Frida官网Gadget介绍地址：`https://frida.re/docs/gadget/`  
比如，GadgetConfig.js中的内容为：
   ```json
    {
      "interaction": {
        "type": "script",
        "path": "/data/local/tmp/xiaojianbang.js"
      }
    }
   ```
   表示让Gadget在app运行时，直接加载hook脚本 `/data/local/tmp/xiaojianbang.js`,
hook脚本路径及名称可自行指定,
Gadget的so存放于 `/system/lib/libxiaojianbang.so` 和 `/system/lib64/libxiaojianbang.so`,
可自行替换版本，支持官网Gadget运行所支持的任意模式

### 3. 整体加固脱壳
打开app，只要dex有加载，就会保存在app的私有目录下
`/data/data/包名/xiaojianbang`。

### 4. 抽取加固脱壳
创建目录 `/data/local/tmp/xiaojianbang/包名/saveDex`，即开启该功能。  
打开app等待一分钟自动开启主动调用，脱壳完成后 logcat中会显示 call run over
将app私有目录下 `/data/data/包名/xiaojianbang` 中的 xxx.dex和xxx.bin 文件拿出来
使用 dexfixer.jar 修复  
    ```shell
    java -jar dexfixer.jar xxx.dex xxx.bin out.dex
    ```  
    out.dex就是最终脱壳完毕的dex。
脱壳完毕，记得删除 `/data/local/tmp/xiaojianbang/包名` 目录，否则每次打开都会脱壳

### 5. 追踪函数调用
需要通过hook开启。
hook libc.so的strstr函数，当参数1为xiaojianbang_javaCall或者xiaojianbang_jniCall时，打印参数0和参数1即可，frida hook代码如下：
```js
function hook_strstr() {
    var libcModule = Process.getModuleByName("libc.so");
    var strstrAddr = libcModule.getExportByName("strstr");
    Interceptor.attach(strstrAddr, {
        onEnter: function (args) {
            this.arg0 = ptr(args[0]).readUtf8String();
            this.arg1 = ptr(args[1]).readUtf8String();
            if (this.arg1.indexOf("xiaojianbang_javaCall") != -1) {
                LogPrint(this.arg1 + "--" + this.arg0);
            }
            if (this.arg1.indexOf("xiaojianbang_jniCall") != -1) {
                LogPrint(this.arg1 + "--" + this.arg0);
            }
        }, onLeave: function (retval) {
             if (this.arg1.indexOf("javaCall") != -1) {
                 retval.replace(1);
             }
             if (this.arg1.indexOf("jniCall") != -1) {
                retval.replace(1);
             }
}
    });
}
```


### 6. smali trace
需要通过hook开启。
hook libc.so的strstr函数，当参数1为shouldTrace时，将函数返回值设置为1，
当参数1为traceLog时，打印参数0和参数1即可，frida hook代码如下：
```javascript
function hook_strstr() {
    var libcModule = Process.getModuleByName("libc.so");
    var strstrAddr = libcModule.getExportByName("strstr");
    Interceptor.attach(strstrAddr, {
        onEnter: function (args) {
            this.arg0 = ptr(args[0]).readUtf8String();
            this.arg1 = ptr(args[1]).readUtf8String();
            if (this.arg1.indexOf("traceLog") != -1) {
               LogPrint(this.arg1 + "--" + this.arg0);
            }
        }, onLeave: function (retval) {
            if (this.arg1.indexOf("shouldTrace") != -1) {
                retval.replace(1);
            }
        }
    });
}
```




