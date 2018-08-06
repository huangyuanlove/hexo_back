---
title: Android中使用WebSocket-2
date: 2017-12-26 16:37:37
tags: [Android,WebSocket]
keywords: Android使用WebSocket
---
上一篇提到在Android中使用WebSocket和服务端进行通信。是直接在Activity里面进行操作的这样会保持一个长连接，一个应用里面没必要也不应该保持多个长连接，所以我们可以把WebSocket客户端挪到Service里面，使用广播和Activity进行通信。
<!-- more -->
1. APP端：
继承`BroadcastReceiver`，重写`public void onReceive(Context context, Intent intent)`方法，在该方法中进行业务处理。
``` java

public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private TextView showMessage;
    private EditText editText;
    private StringBuilder sb = new StringBuilder();
    private WebSocketBroadcastReceiver webSocketBroadcastReceiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent = new Intent(this, WebSocketService.class);
        startService(intent);
        showMessage = findViewById(R.id.show_message);
        editText = findViewById(R.id.edit_text);
        findViewById(R.id.send).setOnClickListener(this);
        webSocketBroadcastReceiver = new WebSocketBroadcastReceiver();
        IntentFilter intentFilter = new IntentFilter("web_socket");
        registerReceiver(webSocketBroadcastReceiver, intentFilter);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.send:

                sb.append("客户端发送消息：");
                sb.append(new Date().toLocaleString());
                sb.append("\n");
                sb.append(editText.getText().toString().trim());
                sb.append("\n");
                showMessage.setText(sb.toString());
                Intent intent = new Intent(this, WebSocketService.class);
                intent.putExtra("message", editText.getText().toString().trim());
                startService(intent);
                editText.setText("");
                break;
            default:
                break;
        }
    }

    class WebSocketBroadcastReceiver extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            String message = intent.getStringExtra("message");
            sb.append("服务端返回消息：");
            sb.append(new Date().toLocaleString());
            sb.append("\n");
            sb.append(message);
            sb.append("\n");
            showMessage.setText(sb.toString());

    }
}

    @Override
    protected void onDestroy() {
        Intent intent = new Intent(this, WebSocketService.class);
        stopService(intent);
        unregisterReceiver(webSocketBroadcastReceiver);
        super.onDestroy();
    }
}
```
2. Service
继承`Service`,在`onCreate()`方法里面创建`WebSocketClient`并和服务端进行连接。在`AndroidManifest.xml`中注册服务。
``` java

public class WebSocketService extends Service {
    private IoTWebSocketClient ioTWebSocketClient;
    private Intent broadcastIntent;

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        broadcastIntent = new Intent();
        broadcastIntent.setAction("web_socket");
        ioTWebSocketClient = new IoTWebSocketClient(URI.create("ws://192.168.1.64:8887"));
        ioTWebSocketClient.connect();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String message = intent.getStringExtra("message");
        if(ioTWebSocketClient.isClosing() || ioTWebSocketClient.isClosed()){
            stopSelf();
            return super.onStartCommand(intent, flags, startId);
        }
        try {
            ioTWebSocketClient.send(message);
        }catch (Exception e){
            e.printStackTrace();
        }
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public void onDestroy() {
        ioTWebSocketClient.close();
        ioTWebSocketClient = null;
        super.onDestroy();
    }

    class IoTWebSocketClient extends WebSocketClient {

        IoTWebSocketClient(URI serverUri) {
            super(serverUri);
        }

        @Override
        public void onOpen(ServerHandshake handshakedata) {
        }

        @Override
        public void onMessage(String message) {
            broadcastIntent.putExtra("message", message);
            WebSocketService.this.sendBroadcast(broadcastIntent);
        }

        @Override
        public void onClose(int code, String reason, boolean remote) {
        }

        @Override
        public void onError(Exception ex) {
            stopSelf();
        }
    }
}
```
在`onStartCommand()`方法里面，对发送消息方法的调用进行异常捕获，是因为这时候可能服务端重启或者服务端还没有准备好，这是发送消息会抛出异常，可以根据自己的业务需求进行改进。

----
以上