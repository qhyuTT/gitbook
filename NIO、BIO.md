NIO、BIO

- 传统的NIO，也就是阻塞IO

  ~~~java
  public static void main(String[] args) throws IOException {
          ServerSocket serverSocket = new ServerSocket(9000);
          while (true) {
              System.out.println("开始连接");
              Socket accept = serverSocket.accept();
              handler(accept);
          }
      }
  
      private static void handler(Socket accept) throws IOException {
          byte[] bytes = new byte[1024];
          System.out.println("准备read。。");
          int read = accept.getInputStream().read(bytes);
          System.out.println("read完毕。。");
          if (read != -1){
              System.out.println("接收到客户端的数据:"+new String(bytes,0,read));
          }
      }
  
  工具使用cmd telnet localhost 9000 ctrl+] 输入send+字符
  ~~~

  

- non-BlockingIO

  ~~~java
  public static void main(String[] args) throws IOException {
          // 创建NIO ServerSocketChannel
          ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
          // 绑定端口
          serverSocketChannel.socket().bind(new InetSocketAddress(9000));
          // 设置ServerSocketChannel为非阻塞，如果为true的话就是NIO
          serverSocketChannel.configureBlocking(false);
          // 添加多路复用器selector，防止空转,操作系统创建一个epoll实例，存放一些监听事件？
          // epoll在liunx中，相对于用list将ServerSocketChannel都保存然后再遍历的方式，epoll能防止空转，windows是selcet使用的是遍历
          Selector selector = Selector.open();
          serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
          System.out.println("服务启动成功！");
          while (true) {
              // 阻塞等待处理的事件发生
              selector.select();
              // 进到后续的代码，说明有事件进入
              // 获取selector中注册的全部事件
              Set<SelectionKey> selectionKeys = selector.selectedKeys();
              Iterator<SelectionKey> iterator = selectionKeys.iterator();
  
              while (iterator.hasNext()) {
                  SelectionKey key = iterator.next();
                  // 处理连接服务端事件
                  if (key.isAcceptable()) {
                      ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                      SocketChannel accept = channel.accept();
                      accept.configureBlocking(false);
                      accept.register(selector, SelectionKey.OP_READ);
                      System.out.println("客户端连接成功!");
                  } else if (key.isReadable()) { // 处理发送send命令事件
                      SocketChannel channel = (SocketChannel) key.channel();
                      ByteBuffer allocate = ByteBuffer.allocate(128);
                      int read = channel.read(allocate);
                      if (read != -1) {
                          System.out.println("接收到消息:" + new String(allocate.array()));
                      } else {
                          System.out.println("客户端断开连接!!!");
                          channel.close();
                      }
                  }
                  // 从事件集合中删除本次处理的事件，防止下次selector重复处理
                  iterator.remove();
              }
          }
      }
  ~~~

  

- what is epoll？epoll模型
  - linux底层创建Selector使用的是epoll模型

- Netty maven配置

  ~~~java
  <dependency>
              <groupId>io.netty</groupId>
              <artifactId>netty-all</artifactId> 
      		<!-- Use 'netty-all' for 4.0 or above -->
              <version>4.1.64.Final</version>
   </dependency>
  ~~~

- 