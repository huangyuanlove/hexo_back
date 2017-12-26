---
title: Android中使用WebSocket
date: 2017-12-25 16:23:46
tags: [Andorid,WebSocket]
---
背景：后端逻辑框架调整，将原来的推送和轮询方式改成了使用`WebSocket`通信。原来的请求方式是由app发起请求，appServer对请求进行分发，中转中继服务器将具体请求下发到对应的物联网服务器，物联网服务器将指令下发到指定的设备。整个流程涉及到很多层http请求，并且每个服务的回调接口还不一致，只能在app发情请求之后，接着去轮询服务器，服务器端去查询设备状态、是否对指令有响应。
改版后涉及到对物联网的请求全部改成`WebSocket`,不在轮询，而是被动等待。
后端使用的是`Spring`实现的`WebSocket`,app端使用的是[https://github.com/TooTallNate/Java-WebSocket](https://github.com/TooTallNate/Java-WebSocket)这个开源项目。
<!-- more -->
#### APP端实现
1. 添加依赖`compile "org.java-websocket:Java-WebSocket:1.3.7"`
2. 我们只需要关心三方库中`WebSocketClient`类就可以了，其他细节底层已经封装好了。
3. 类中有四个方法需要重写：
``` java
/**打开连接*/
 public void onOpen(ServerHandshake handshakedata)
 /**服务端返回消息*/
 public void onMessage(String message) 
 /**关闭连接*/
 public void onClose(int code, String reason, boolean remote)
 /**出现异常*/
 public void onError(Exception ex)
```
4. 一个简单的小测试，app端定义了一个发送按钮，和一个展示消息的文本
``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.huangyuanlove.testwebsocket.MainActivity">

    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1">

        <TextView
            android:id="@+id/show_message"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" />
    </ScrollView>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal">

        <EditText
            android:id="@+id/edit_text"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1" />

        <TextView
            android:id="@+id/send"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:background="@color/colorAccent"
            android:padding="10dp"
            android:text="发送"
            android:textColor="@android:color/black" />
    </LinearLayout>

</LinearLayout>
```
5. 在代码里面处理具体逻辑：
``` java
import android.os.Bundle;
import android.os.Handler;
import android.os.Message;
import android.support.design.widget.Snackbar;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.EditText;
import android.widget.TextView;

import org.java_websocket.client.WebSocketClient;
import org.java_websocket.handshake.ServerHandshake;

import java.net.URI;
import java.util.Date;

public class MainActivity extends AppCompatActivity implements View.OnClickListener {


    private TextView showMessage;
    private EditText editText;
    private WebSocketClient webSocketClient;
    private StringBuilder sb = new StringBuilder();

    private Handler handler = new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            sb.append("服务器返回数据：");
            sb.append(msg.obj.toString());
            sb.append("\n");
            showMessage.setText(sb.toString());
            return true;
        }
    });

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        showMessage = findViewById(R.id.show_message);
        editText = findViewById(R.id.edit_text);
        findViewById(R.id.send).setOnClickListener(this);
        URI serverURI = URI.create("ws://192.168.1.199:8887");
        webSocketClient = new WebSocketClient(serverURI) {
            @Override
            public void onOpen(ServerHandshake handshakedata) {
                sb.append("onOpen at time：");
                sb.append(new Date());
                sb.append("服务器状态：");
                sb.append(handshakedata.getHttpStatusMessage());
                sb.append("\n");
                showMessage.setText(sb.toString());
            }
            @Override
            public void onMessage(String message) {
                Message handlerMessage = Message.obtain();
                handlerMessage.obj = message;
                handler.sendMessage(handlerMessage);
            }
            @Override
            public void onClose(int code, String reason, boolean remote) {
                sb.append("onClose at time：");
                sb.append(new Date());
                sb.append("\n");
                sb.append("onClose info:");
                sb.append(code);
                sb.append(reason);
                sb.append(remote);
                sb.append("\n");
                showMessage.setText(sb.toString());
            }

            @Override
            public void onError(Exception ex) {
                sb.append("onError at time：");
                sb.append(new Date());
                sb.append("\n");
                sb.append(ex);
                sb.append("\n");
                showMessage.setText(sb.toString());
            }
        };
        webSocketClient.connect();


    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.send:
                if(webSocketClient.isClosed() || webSocketClient.isClosing()){
                    Snackbar.make(v,"Client正在关闭",Snackbar.LENGTH_SHORT).show();
                    webSocketClient.connect();
                    break;
                }
                webSocketClient.send(editText.getText().toString().trim());
                sb.append("客户端发送消息：");
                sb.append(new Date());
                sb.append("\n");
                sb.append(editText.getText().toString().trim());
                sb.append("\n");
                showMessage.setText(sb.toString());
                editText.setText("");
                break;
            default:
                break;
        }
    }
}

```
#### 服务端实现
上面提到我们后端使用的Spring中的WebSocket实现，其实用什么实现服务端无所谓，只要遵循协议就可以。个人在本地做测试的时候用的还是这个开源项目。
1. 服务端只需要关心`WebSocketServer`这个类就好，这个类里面有五个方法需要重写：
``` java
/**服务开启*/
public void onStart() 
/**有客户端连接*/
public void onOpen(WebSocket webSocket, ClientHandshake clientHandshake)
/**服务端关闭*/
public void onClose(WebSocket webSocket, int i, String s, boolean b)
/**收到客户端的消息*/
public void onMessage(WebSocket webSocket, String s)
/**出现异常*/
 public void onError(WebSocket webSocket, Exception e) 
```
2. 具体代码如下：
``` java
package com.huangyuanlove;

import org.java_websocket.WebSocket;
import org.java_websocket.WebSocketImpl;
import org.java_websocket.handshake.ClientHandshake;
import org.java_websocket.server.WebSocketServer;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.InetSocketAddress;
import java.net.UnknownHostException;

public class TestWebSocket extends WebSocketServer {
    public TestWebSocket(int port) throws UnknownHostException {
        super(new InetSocketAddress(port));
    }

    public TestWebSocket(InetSocketAddress address) {
        super(address);
    }

    public static void main(String[] args) {
        WebSocketImpl.DEBUG = true;
        try {
            int port = 8887; // 843 flash policy port
            TestWebSocket s = new TestWebSocket(port);
            s.start();
            System.out.println("ChatServer started on port: " + s.getPort());

            BufferedReader sysin = new BufferedReader(new InputStreamReader(System.in));
            while (true) {
                String in = sysin.readLine();
                s.broadcast(in);
                if (in.equals("exit")) {
                    s.stop(1000);
                    break;
                }
            }
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }

    @Override
    public void onOpen(WebSocket webSocket, ClientHandshake clientHandshake) {

        broadcast("new connection: " + clientHandshake.getResourceDescriptor());
        System.out.println(webSocket.getRemoteSocketAddress().getAddress().getHostAddress() + " entered the room!");
    }

    @Override
    public void onClose(WebSocket webSocket, int i, String s, boolean b) {
        broadcast(webSocket + " onClose");
        System.out.println(webSocket + " onClose");
    }

    @Override
    public void onMessage(WebSocket webSocket, String s) {

        broadcast(s);
        System.out.println(webSocket + ": " + s);
    }

    @Override
    public void onError(WebSocket webSocket, Exception e) {
        e.printStackTrace();
        if (webSocket != null) {
            // some errors like port binding failed may not be assignable to a specific websocket
        }
    }

    @Override
    public void onStart() {
        System.out.println("Server started!");
    }
}
```
上面代码中`onMessage`方法中的`broadcast`方法是向所有连接到服务器的客户端发送消息(广播发送，其实就是一个小型的局域网聊天室)，如果只是谁发来的消息就回复给谁，可以调用`webSocket.send()`方法。
是用的时候先开启服务端，然后开启客户端(app)，需要注意是的，在客户端中重写的方法都不是在主线程中，如果需要更新UI，请切换到UI线程。
或者在客户端中使用`Service`,在`Service`中收到消息之后，广播给UI界面。

----
以上