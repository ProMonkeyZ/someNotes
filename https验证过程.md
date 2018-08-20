# https验证过程
## https:
* https 就是HTTP Over SSL. 是客户端和服务器之间的一个数据传输协议.
* 对称加密是指加密用的公钥和私钥都相同的加密方式,例如凯撒加密算法.
* 非对称加密指加密用的公钥和私钥不同的加密方式.

##### 对称加密

对称加密的加密和解密过程可以认为是一个互逆的过程.即加密秘钥可以从解密秘钥中推算出来,同样的解密秘钥也可以从加密秘钥中推算出来.所以大多数的对称加密算法的加密秘钥和解密秘钥都是相同的.

##### 非对称加密

非对称加密即指用一对加密秘钥对信息进行加密,私钥加密公钥解密,公钥加密私钥解密.非对称加密使用公钥进行加密的时候速度较慢,如果要加密大量数据使用非对称加密效率太低.
公钥只能用作加密秘钥,而私钥只能用作解密秘钥. 

> RSA算法是很有名的非对称加密算法

##### 整个使用数字签名进行一次消息发送

###### 客户端给服务器发送信息:

> (content+pri)发送给服务器,然后服务器使用pri解密.

###### 服务器给客户端回信:

> content+hash=>摘要(digest)

> digest+pri=>签名(signature)

> content+signature=>一封正式的回信.

###### 客户端收到服务器的回信:

> 收信后使用pub对signature进行解密,得到digest_receive(摘要).然后对content使用同样的hash再得到一份digest_selfBuild(摘要).如果

```
digest_receive == digest_selfBuild
```
则证明content未被修改过,并且数据确实是由目标服务器发出来的.

###### CA机构,CA证书:

> CA_pri+pub+service_info=CA_Digital_Certificate(CA证书).

###### 真正的数据传递过程:

> content+signature+CA_Digital_Certificate=>一封正式的服务端回信.
> 客户端收到回信之后:

```
1. 使用CA_pub对CA_Digital_Certificate解密,得到服务端的pub.
2. 使用使用上一步得到的pub对回信中的signature进行解密,如果可以解密说明该公钥是正确的服务端公钥.并且得到摘要.
3. 使用同样的hash算法对content进行计算得到摘要.
4. 回到之前说的如果两次得到的摘要是相同的,那么说明content未被修改过.
```
##### 实际应用
所以在实际应用中我们使用非对称加密管理对称加密的秘钥,然后使用对称加密算法加密数据,这样就可以实现既安全又快速的传输数据.为了使对称加密的秘钥不被劫持或者发现,我们使用上述带CA认证的发送方法在http请求的第一次握手建立时传输对称加密要使用的秘钥.

```
1. client发起三次握手
2. service返回CA证书给client
3. client用下载好的CA根证书用包含的CA.pub进行解密,得到service.pub;验证CA证书是合法证书.
4. client生成一个随机对称秘钥,使用上一步得到的service.pub进行加密然后发送给服务器.
5. 接下来client和service利用第4步生成的对称秘钥进行http通信.通信过程中使用对称加密对收发的信息进行加解密.
```
![httpsCreat.png](https://i.loli.net/2018/08/01/5b615e9ae193d.png)
