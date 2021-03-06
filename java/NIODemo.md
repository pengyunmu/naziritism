### 客户端代码
```
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;

/**
 * Created by MuPengYun on 2016/8/25.
 *
 * NIO demo 客户端程序
 */
public class TCPClient {

    // 服务端的IP
    private String remoteServerIp;
    // 服务端监听的端口
    private int remoteServerPort;
    // 多路复用器
    private Selector selector;
    // 通信信道
    SocketChannel socketChannel;

    /**
     * 构造函数
     * @param remoteServerIp
     * @param remoteServerPort
     * @throws IOException
     */
    public TCPClient(String remoteServerIp, int remoteServerPort) throws IOException {
        this.remoteServerIp = remoteServerIp;
        this.remoteServerPort = remoteServerPort;

        initialize();
    }
    /**
     * 初始化socketChannel
     * @throws IOException
     */
    private void initialize() throws IOException {
        // 打开信道，并设置为非阻塞模式
        socketChannel = SocketChannel.open();
        socketChannel.connect(new InetSocketAddress(remoteServerIp,remoteServerPort));
        socketChannel.configureBlocking(false);

        // 打开信道选择器,并注册选择器到信道
        selector =  Selector.open();
        socketChannel.register(selector, SelectionKey.OP_READ);

        // 启动客户端读线程
        new clientRead(selector).start();

    }
    public void sendMsg (String message) throws IOException {
        ByteBuffer byteBuffer = ByteBuffer.wrap(message.getBytes("UTF-8"));
        socketChannel.write(byteBuffer);
        byteBuffer.clear();
    }
    public static void main(String args[]) throws Exception {
        TCPClient tcpClient = new TCPClient("localhost",12345);
        tcpClient.sendMsg("Hi 服务端，我是你的仆人小美，有什么需要帮助的吗？");
    }
}

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.nio.charset.Charset;


/**
 * Created by MuPengYun on 2016/8/25.
 */
public class clientRead extends Thread{

    private Selector selector;

    public clientRead(Selector selector){
        super();
        this.selector = selector;

    }
    // byteBuffer
    @Override
    public void run() {

        try {
            while(selector.select()>0){
                // 遍历每个有可用IO操作Channel对应的SelectionKey
                for(SelectionKey selectionKey : selector.selectedKeys()){
                    // 如果该SelectionKey对应的Channel中有可读的数据
                    if(selectionKey.isReadable()){
                        // 使用NIO读取Channel中的数据
                        SocketChannel socketChannel = (SocketChannel)selectionKey.channel();
                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                        // 使用byteBuffer 独处Channel中的数据
                        if(socketChannel.read(byteBuffer) > 0)
                        {
                            byteBuffer.flip();

                            // 字节转化utf-8的字符串
                            String readStringFromServer = Charset.forName("UTF-8").newDecoder().decode(byteBuffer).toString();

                            System.out.println("接收到来自服务器"+socketChannel.socket().getRemoteSocketAddress()+"的信息:"+readStringFromServer);
                            byteBuffer.clear();

                        }else{
                            System.out.println("没有读取到信息");
                        }


                    }else{
                        // 什么也不做
                    }
                    selector.selectedKeys().remove(selectionKey);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
###服务器代码
```
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.nio.charset.Charset;

/**
 * Created by MuPengYun on 2016/8/25.
 * NIO demo 服务端程序
 */
public class TCPServer {

    // 本地服监听的端口
    private static final int ListenPort = 12345;

    public static void main(String args[])  {
        // 开启监听信道并绑定监听端口
        ServerSocketChannel serverSocketChannel = null;
        try {
            serverSocketChannel = ServerSocketChannel.open();

        serverSocketChannel.bind(new InetSocketAddress(ListenPort));

        // 设置非阻塞模式
        serverSocketChannel.configureBlocking(false);
        // 开启选择器
        Selector selector = Selector.open();
        // 在信道中注册选择器 只有非阻塞信道才可以注册选择器.并在注册过程中指出该信道可以进行Accept操作
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while(true){
            // 当信道可用时 每次连接一个select
            if(selector.select(3000) == 0){
                System.out.println("我现在空闲，快来连接我啊！！！");
            }
            for(SelectionKey selectionKey : selector.selectedKeys()){
                if(selectionKey.isAcceptable()){
                    SocketChannel socketChannel = ((ServerSocketChannel)selectionKey.channel()).accept();
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

                    while(socketChannel.read(byteBuffer) > 0){
                        // 缓存区准备为数据传出状态
                        byteBuffer.flip();

                        // 字节转化utf-8的字符串
                        String receivedString = Charset.forName("UTF-8").newDecoder().decode(byteBuffer).toString();

                        System.out.println("接收到来自客户端"+socketChannel.socket().getRemoteSocketAddress()+"的信息:"+receivedString);
                        byteBuffer.clear();
                        // 返回客户端信息
                        String sendMesg = "Hi客户端你好，我知道你是小美，你去给我拖个厕所吧！！！";

                        ByteBuffer response=ByteBuffer.wrap(sendMesg.getBytes("UTF-8"));

                        socketChannel.write(response);
                        response.clear();
                    }
                }else{
                   // donothing
                }
                selector.selectedKeys().remove(selectionKey);
            }

        }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
