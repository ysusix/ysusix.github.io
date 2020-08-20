## 网络模型
多路复用
```
// ae.h / ae.c / ae_epooll.c / ae_evport.c / ae_kqueue.c / ae_select.c

```
https://www.jianshu.com/p/ee381d365a29

## RESP Redis Serialization Protocol
- 单行字符串以 + 符号开头，如 +hello world\r\n
- 多行字符串以 $ 符号开头，后跟字符串长度，如 $11\r\nhello world\r\n
- 整数值以 : 符号开头，后跟整数的字符串形式，如 :1024\r\n
- 错误消息以 - 符号开头，如 -WRONGTYPE Operation against a key holding the ...
- 数组以 * 符号开头，后跟数组长度，如 *3\r\n:1\r\n:2\r\n:3\r\n
- NULL: $-1\r\n
- 空字符串: $0\r\n\r\n
- set author codehole: *3\r\n$3\r\nset\r\n$6\r\nauthor\r\n$8\r\ncodehole\r\n
- OK: +OK