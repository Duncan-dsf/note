- channel 与buffer 先从channel中读到buffer中，对buffer进行读写
- channel通过ServerSocketChannel.open()获取，获取到之后需要监听一个端口
- 之后accept()等待客户端链接,客户端链接之后 返回socketChannel对象



### channel与buffer

- channel.read(buffer)
- 从channel中读取数据到buffer中

### ServerSocketChannel

- accept是否阻塞
- selector如何使用