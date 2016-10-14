# usb同屏控制

## 



### 一、场景

* 定制机批量安装apk，使用脚本批量设置权限。
* pad教学过程中，老师端实时显示学生pad屏幕、及时提供指导。
* [真机远程调试](http://mtc.baidu.com/tinypace/main#/?from=DaoHang)


### 二、vysor

* 收费、免费版不清晰
* 只能单控，不能多控



### 三、演示

![](http://raw.githubusercontent.com/gackor/usbcontrol/master/us2.gif)
![](http://raw.githubusercontent.com/gackor/usbcontrol/master/sc1.gif)

### 四、截屏常见的方案

#### 

 * View.getDrawingCache()
 
    这是最常见的应用内截屏方法，这个函数的原理就是通过view的Cache来获取一个bitmap对象，然后保存成图片文件，这种截屏方式非常的简单，但是局限行也很明显，首先它只能截取应用内部的界面，甚至连状态栏都不能截取到。其次是对某些view的兼容性也不好，比如webview内的内容也无法截取。


* 读取/dev/graphics/fb0

    因为Android是基于linux内核，所以我们也能在android中找到framebuffer这个设备，我们可以通过读取/dev/graphics/fb0这个帧缓存文件中的数据来获取屏幕上的内容，但是这个文件是system权限的，所以只有通过root才能读取到其中的内容，并且直接通过framebuffer读取出来的画面还需要转换成rgb才能正常显示。

* 反射调用SurfaceControl.screenshot()/Surface.screenshot()

    SurfaceControl.screenshot()(低版本是Surface.screenshot())是系统内部提供的截屏函数，但是这个函数是@hide的，所以无法直接调用，需要反射调用。我尝试反射调用这个函数，但是函数返回的是null，后面发现SurfaceControl这个类也是隐藏的，所以从用户代码中无法获取这个类。也有一些方法能够调用到这个函数，比如重新编译一套sdk，或者在源码环境下编译apk，但是这种方案兼容性太差，只能在特定ROM下成功运行。

* screencap -p xxx.png/screenshot xxx.png
    
    这两个是在shell下调用的命令，通过adb shell可以直接截图，但是在代码里调用则需要系统权限，所以无法调用。可以看到要实现类似vysor的同步操作，可以使用这两个命令来截取屏幕然后传到电脑显示，但是我自己实现后发现这种方式非常的卡，因为这两个命令不能压缩图片，所以导致获取和生成图片的时间非常长。

* MediaProjection,VirtualDisplay (>=5.0)

    在5.0以后，google开放了截屏的接口，可以通过”虚拟屏幕”来录制和截取屏幕，不过因为这种方式会弹出确认对话框，并且只在5.0上有效。


--------
### 五、本次开发采用方案

1、截屏方案
```java
    public static Bitmap screenShoot(int w, int h) throws Exception{
        Class sc = Class.forName("android.view.SurfaceControl");
        Method method=sc.getMethod("screenshot",
                       new Class[] {int.class, int.class});
        Object o = method.invoke(sc, new Object[]{w,h});
        return (Bitmap)o;
    }

```

2、实现架构

* android 
```java
class ScreenWebSocketServer extends WebSocketServer {
    public ScreenWebSocketServer(int port) {
        super(new InetSocketAddress(port));
    }
}
public static void main(String[] args) {
	int port=Integer.parseInt(args[1]);
	ScreenWebSocketServer server=new ScreenWebSocketServer(port);
	server.start();
}
```
```java
	/**执行服务端发过来的鼠标事件*/
    public static void injectMotionEvent(InputManager paramInputManager, Method paramMethod, long downTime, int action, float x, float y) {
        MotionEvent localMotionEvent = MotionEvent.obtain(downTime, downTime, action, x, y, 0);
        localMotionEvent.setSource(InputDevice.SOURCE_TOUCHSCREEN);
        try {
            paramMethod.invoke(paramInputManager, new Object[]{localMotionEvent, Integer.valueOf(0)});
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
	/**执行服务端发过来的命令事件*/
    public static String execCommand(String command) {
        Runtime runtime = Runtime.getRuntime();
        Process proc = null;
        try {
            proc = runtime.exec(command);
            if (proc.waitFor() != 0) {
                System.err.println("exit value = " + proc.exitValue());
            }
            BufferedReader in = new BufferedReader(new InputStreamReader(
                    proc.getInputStream()));
            StringBuffer stringBuffer = new StringBuffer();
            String line = null;
            while ((line = in.readLine()) != null) {
                stringBuffer.append(line+" ");
            }
            return stringBuffer.toString();
        } catch (Exception e) {
            System.err.println(e);
        }finally{
            try {
                proc.destroy();
            } catch (Exception e2) {
            }
        }
        return "";
    }



```
* server #node.js
```javascript
/**创建socket与app通信*/
    function createSocket(port){
		var wsUrl = 'ws://localhost:'+port; //服务器地址
		var websocket = new WebSocket(wsUrl); //创建WebSocket对象
		websocket.onopen = function (evt) {
			console.log(port+":已经建立连接"+evt.message);
			websocket.send("run");
		};
		websocket.onclose = function (evt) {console.log(port+":连接关闭"+evt.message);};
		websocket.onerror = function (evt) {console.log(port+":"+evt.message);};
        //
        websocket.onmessage = function (evt) {
			var index=port-18880
			//收到服务器消息，使用evt.data提取
			htmlSocketArray[index].send(evt.data);
		};
		return websocket;
	}
 ```
 ```javascript
/**创建websocket服务器与前端通信*/
    var WebSocket=require('ws')
	var htmlSocketArray=new Array();
	var htmlServer = new (require('ws').Server)({port: 18999});
	htmlServer.on('connection', function(socket) {
		htmlSocketArray.push(socket);
		func.changeHtmlSocket(htmlSocketArray);
		console.log( 'New HtmlClient Connection ('+htmlServer.clients.length+' total)' );
		socket.on('close', function(code, message){
			console.log( 'Disconnected HtmlClient ('+htmlServer.clients.length+' total)' );
		});
		if(htmlSocketArray.length==1){
			socket.on('message',function(message){
				console.log(message);
				dealMsg(message);
			});
		}

	});
    
```
```javascript
/**命令执行*/
exports.adbDevices =function (apkPath){
		var stdou=execSync("adb devices",{encoding:'utf8'}).replace(/device/g,"");
		var arr=stdou.match(/\b(\w+)\b/g);
		if(arr.length<4)return;
		console.log("current arrs:"+arr);
		var port=18880;
		var index=0;
		for (var i = 4; i < arr.length; i++) {
			console.log("current arr:"+arr[i]);
			if (deviceArray.indexOf(arr[i]) == -1) {
				deviceArray.push(arr[i]);
				var cmd1="adb -s "+arr[i]+" forward tcp:"+port+" tcp:"+port;
				var forward=execSync(cmd1);
				console.log(cmd1+"执行结果:success");
				var cmd2="adb -s "+arr[i]+" uninstall com.gackor";
				var uninstall=execSync(cmd2,{encoding:'utf8'});
				console.log(cmd2+"执行结果:"+uninstall);
				var install=execSync("adb -s "+arr[i]+" install "+apkPath,{encoding:'utf8'});
				console.log("install执行结果:port:"+port+"  ==="+install);
				var imeset=execSync("adb -s "+arr[i]+" shell ime set com.gackor/com.AdbIME",{encoding:'utf8'});
				console.log("set inputMethod执行结果："+imeset);
				var shell=exec("adb -s "+arr[i]+" shell \"export CLASSPATH=/data/app/com.gackor-1/base.apk&&exec app_process /system/bin demo.ChatServer '$@' "+port+"\"");
				shell.stdout.on('data', function(data) {
					if(data.trim()!="") {
						if (data.indexOf("socketServer-started")!=-1) {
							appSocketArray.push(createSocket(18880+index));
							index=index+1;
						}

					}
				});
				port=port+1;
			}

		}
	};
	//
	exports.inputMsg=function (message) {
		for(var d=0;d<deviceArray.length;d++){
			//
			execSync("adb -s "+deviceArray[d]+" shell am broadcast -a ADB_INPUT_TEXT --es msg "+message);
		}

	}

```

* 前端
 ```javascript
        function createSocket(flag){
            var websocket = new WebSocket(wsServer);
            websocket.onopen = function (evt) {writeToScreen(flag+":已经建立连接");};
            websocket.onclose = function (evt) {writeToScreen(flag+":连接关闭");};
            websocket.onerror = function (evt) {writeToScreen(flag+evt.message);};
            websocket.onmessage = function (evt) {
                if(start==false){
                    writeToScreen("收到:"+evt.data)
                    start=drawCanvas(evt.data);
                    if(start){
                        sendMsg("run");
                    }
                    return;
                }
                if(flag==0){
                    canvas.src = URL.createObjectURL(evt.data);
                }else if(flag==1){
                    canvas2.src = URL.createObjectURL(evt.data);
                }else if(flag==2){
                    canvas3.src = URL.createObjectURL(evt.data);
                }

            };
            return websocket;
//
        };
```


### 六、继续开发
1、随意切换单控

2、支持录制脚本

3、h264视频流编码


---

### 七、下周计划：

android热更新：不需要安装、不需要重启app即可在线修复bug
