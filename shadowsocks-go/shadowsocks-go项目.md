

# shadowsocks-go项目

## 主流程



## 关键点

- ### socks5协议交互


1. client与local-server建立tcp连接后，client发送request来协商版本和认证方式

   ![](image/ver_auth_req.png)

   - ver：0x05

   - nmethods: 客户端支持的方法总数

   - methods:客户端支持的方法，每个方法标志占一个字节

2. local-server给client的返回

   ![](image/ver-auth-resp.png)

   - ver: 0x05
   - method: 0x00 表示没有启用认证

3. client发送connect request给local-server

   ![](image/conn_req.png)

   - ver : 0x05
   - cmd:0x01 -->sock CONNECT命令
   - ATYP：地址类型
     1. IPV4地址: X’01’
     2. 域名地址: X’03’ ，该情况下，DST.ADDR中有一个addrlen字段占一个字节，表示域名地址的长度
     3. IPV6地址: X’04’

4. local-server给client的返回

   ![](image/conn_resp.png)

   - ver：0x05

   - rep: 0x00-->成功

   - atyp:0x01--> IPV4 address

   - BND.ADDR: 服务器绑定的地址，0X00 0X00 0X00 0X00

   - BND.PORT: 服务器绑定的端口；


- ### leakybuf 内存池实现

  ```go
  type LeakyBuf struct {
  	bufSize  int // size of each buffer
  	freeList chan []byte
  }
  
  const leakyBufSize = 4108 // data.len(2) + hmacsha1(10) + data(4096)
  const maxNBuf = 2048
  
  var leakyBuf = NewLeakyBuf(maxNBuf, leakyBufSize)
  ```

  1. 属性
     - buffer : []byte,大小由bufSize指定
     - bufSize :每个buffer的大小
     - freeList: 用于缓存buffer

  2. 方法

     - get方法： 获取buffer,当缓存中没有可用buffer时，创建新的buffer

     ```go
     func (lb *LeakyBuf) Get() (b []byte) {
     	select {
     	case b = <-lb.freeList:
     	default:
     		b = make([]byte, lb.bufSize)
     	}
     	return
     }
     ```

     - put方法：将buffer放到缓存中,当缓存中满时，不做处理，让其内存自动回收

       ```go
       func (lb *LeakyBuf) Put(b []byte) {
       	if len(b) != lb.bufSize {
       		panic("invalid buffer size that's put into leaky buffer")
       	}
       	select {
       	case lb.freeList <- b:
       	default:
       	}
       	return
       }
       ```


- ### 连接错误处理

