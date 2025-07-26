### 加密算法

* **对称加密**：加密解密使用相同密钥，典型的算法示例为AES
* **非对称加密**：存在一对秘钥，公钥加密，私钥解密(加密场景)，私钥签名，公钥验签(签名场景)，典型的算法示例为RSA

| 特性     | 对称加密         | 非对称加密         |
| :------- | :--------------- | :----------------- |
| **速度** | ⚡ 极快（GB/s级） | 🐢 慢（约慢1000倍） |

### 为什么需要 HTTPS？

1. **HTTP 的问题：明信片时代**
   - **明文传输：** HTTP 协议传输的所有数据（你浏览的网页内容、输入的账号密码、信用卡号、聊天记录等）都是未经加密的明文。就像写在明信片上。
   - **窃听风险：** 数据在网络中传输会经过很多节点（路由器、交换机、WiFi 接入点等）。任何一个节点上的攻击者都可以轻易**窃听**并获取你传输的敏感信息。
   - **篡改风险：** 攻击者不仅可以看，还可以**篡改**你发送或接收的数据。例如，在网页里插入广告、恶意链接，或者修改你转账的收款账号和金额（中间人攻击）。
   - **冒充风险：** 你访问的 `www.yourbank.com` 真的是你的银行吗？攻击者可以搭建一个一模一样的假网站（钓鱼网站），诱骗你输入账号密码。HTTP 本身无法验证网站的真实身份。
2. **HTTPS 的诞生：安全信封**
   - **核心目标：** 解决 HTTP 的三大安全问题 - **保密性、完整性、身份认证**。
   - **基本定义：** HTTPS 不是一个新的协议。它 = **HTTP** + **SSL/TLS**。
     - `HTTP`：负责实际的数据传输（内容本身）。
     - `SSL (Secure Sockets Layer)` / `TLS (Transport Layer Security)`：负责在 HTTP 传输层之上建立一个**安全的加密通道**。目前主要使用更安全、更新的 TLS (SSL 3.0 已被淘汰，通常我们说 SSL 指代的是 TLS)。

### HTTPS 如何工作？（TLS/SSL 的作用）

HTTPS 的安全保障主要依赖于 TLS/SSL 协议。其核心过程是 **“TLS 握手”**，目的是协商出一个只有客户端和服务器知道的**会话密钥**，用于后续通信的对称加密。**握手过程中使用到了非对称加密，在传输数据时使用对称加密。**

- **密钥交换简化流程：**

  1. 服务器把自己的**公钥**发送给客户端（包含在证书中）。
  2. 客户端生成一个随机的**对称会话密钥**。
  3. 客户端用服务器的**公钥**加密这个**对称会话密钥**。
  4. 客户端把加密后的对称密钥发送给服务器。
  5. 服务器用自己的**私钥**解密，得到对称会话密钥。
  6. 现在双方都安全地拥有了同一个对称会话密钥，后续通信使用它进行快速的对称加密。

- **为什么不能只用对称加密？** 

  因为如何安全地把这个对称密钥从服务器传给客户端是个大问题（在互联网上直接发密钥，会被窃听）。

- **为什么需要证书，服务器直接发送公钥，不通过证书行不行?**

  没有证书的情况下，如何确保拿到的公钥就是我想要访问的网站发送的呢，而不是攻击者伪造的，**解决方案就是通过数字证书，因为证书上有签名，浏览器必须验证签名成功后，才会信任这个证书，信任这个公钥。**

- **证书内容：** 由受信任的第三方机构 **CA (Certificate Authority)** 颁发。包含：

  - 网站的域名 (Subject)。
  - 网站服务器的**公钥**。
  - 证书的颁发者 (Issuer - CA)。
  - 证书的有效期。
  - CA 使用自己的**私钥**对以上信息做的**数字签名**。

- **验证过程 (信任链)：**

  1. 客户端（浏览器/操作系统）内置了**信任的根 CA 证书(包含了对应公钥)。**
  2. 服务器在握手时发送我们自己网站的数字证书文件(**该证书由CA机构颁发，并且使用了CA机构的私钥进行签名**)。
  3. 客户端用内置的根 CA 公钥验证证书上 CA 的签名是否有效。
  4. 客户端检查证书中的域名是否与当前访问的域名一致。
  5. 客户端检查证书是否在有效期内。
  6. 如果所有检查都通过，客户端就**信任**这个证书，进而信任里面的公钥确实属于它正在访问的合法网站。

- **CA 的角色：** 充当互联网上的“公证处”。它会在签发证书前验证申请者（通常是网站所有者）对该域名的控制权（验证过程严格程度不同，对应 DV, OV, EV 证书等级）。浏览器只信任由它内置列表中的 CA 签发的证书。

- **锁图标的意义：** 浏览器显示锁图标，意味着它成功验证了服务器的证书，确认了网站的身份。

- **如果浏览器无法验证网站的证书时会发生什么？**

  此时浏览器会将访问控制权交给用户，并且地址栏会显示红色的叉号，如果用户确认要访问时，数据传输过程还是会进行加密，因为数字证书上的公钥可以直接获取到，用来对对称加密使用到的密钥进行加密，只不过该公钥(证书)不被浏览器信任而已。

### 自签名证书

所谓自签名证书就是用自己的私钥来进行签名的，即网站服务端使用这个证书时，只要操作系统也安装了这个证书，使用https访问时就不会有安全警告了。不像CA机构颁发的证书，是使用CA机构的私钥进行签名的，因此需要操作系统安装该CA机构私钥对应的证书文件(证书包含了对应的公钥)，这样浏览器才能对自己的证书进行签名验证。

### 证书内容格式

| 格式        | 特点                                             | 典型扩展名     |
| :---------- | :----------------------------------------------- | :------------- |
| **PEM**     | 文本格式，Base64编码，二进制格式的Base64文本表示 | `.pem`, `.crt` |
| **DER**     | 二进制格式                                       | `.der`, `.cer` |
| **PKCS#12** | 加密容器，一个打包文件（打包证书+私钥）          | `.p12`, `.pfx` |

典型PEM文件示例

证书文件

```
-----BEGIN CERTIFICATE-----
MIIFazCCA1OgAwIBAgIRAIIQz7DSQONZRGPgu2OCiwAwDQYJKoZIhvcNAQELBQAw
...（Base64编码数据）...
-----END CERTIFICATE-----
```

私钥

```
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDfX4sJqD7v7S4d
...（Base64编码数据）...
-----END PRIVATE KEY-----
```

证书签名请求(CSR)

```
-----BEGIN CERTIFICATE REQUEST-----
MIIEkjCCAnoCAQAwFjEUMBIGA1UEAwwLZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3
...（Base64编码数据）...
-----END CERTIFICATE REQUEST-----
```

### PKCS#12

PKCS#12（**Public Key Cryptography Standards #12**）是一种**文件格式标准**，用于安全地存储和传输加密的私钥及其关联的数字证书（如X.509证书）。它通常用于打包用户的个人身份凭证（私钥+证书+信任链），便于在不同系统或应用之间迁移或备份。

#### 核心概念：

1. **目的**：
   **打包私钥与证书**，确保其机密性和完整性，方便传输或备份。
2. **文件扩展名**：
   - `.p12` 或 `.pfx`（两者通常可互换使用，但严格来说PFX是PKCS#12的前身）。
3. **核心内容**：
   - **私钥**（加密存储）
   - **数字证书**（用户证书）
   - **证书链**（可选，包含中间CA证书）
   - 其他属性（如友好名称、本地密钥ID等）
4. **安全机制**：
   - 使用**密码（口令）** 加密整个文件（基于口令的加密，PBE）。
   - 支持强加密算法（如AES、3DES），防止未授权访问。
5. **典型应用场景**：
   - 在Web浏览器中**导入/导出SSL/TLS客户端证书**（如网银U盾）。
   - 为服务器（如Apache/Nginx）**配置HTTPS证书**（私钥+证书打包）。
   - 移动设备或邮件客户端（如Outlook）的证书部署。
   - Java Keystore（JKS）的替代方案（`keytool`支持PKCS#12）。
6. **生成pkcs#12文件，使用java提供的keytool命令，当然也可以使用openssl**

```bash
# 生成
keytool -genkeypair -alias wastonl -dname "CN=www.wastonl.com" \
-ext "SAN=dns:www.wastonl.com,dns:localhost,ip:127.0.0.1" \
-keyalg RSA -keysize 2048 \
-storetype PKCS12  -keystore wastonl.p12 -storepass 123456 \
-validity 3650

# 查看详情(敲回车后，会提示输入密码才能查看，生成时-storepass指定的密码)
keytool -list -v -keystore wastonl.p12 -storetype PKCS12

# 导出其中的证书文件，keytool命令不直接支持导出私钥文件，-rfc导出为pem格式，不加则是二进制格式
# 同样要输入密码
keytool -exportcert -alias wastonl -file wastonl.crt -rfc \
-keystore wastonl.p12 -storetype PKCS12
```

**参数说明**：

- `-alias`：密钥条目别名（自定义名称）
- `-dname`: 证书主题，同下面openssl的-subj选项
- `-ext`：添加扩展，同下面openssl的-addext选项
- `-keyalg`：密钥算法（推荐 RSA 或 EC）
- `-keysize`：密钥长度
- `-storetype`：指定密钥库格式为 PKCS12，从JDK9开始默认为PKCS12，JDK8是JKS
- `-keystore`：输出文件名（建议使用 `.p12` 扩展名）
- `-validity`：证书有效期（天）
- `-storepass`：密钥库密码（此处为示例，请替换）

7. **pkcs#12文件和pem格式证书一样，支持安装到操作系统中，因为内部包含证书文件**

### https示例

使用自签名证书 + SpringBoot构建

#### 使用openssl创建自签名证书(pem格式)

```bash
# 一步直接生成私钥以及证书文件
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout private.key -out wangtaoj.crt \
  -subj "/C=CN/ST=Hunan/L=Changsha/O=wangtaoj/OU=wangtaoj/CN=www.wangtaoj.com" \
  -addext "subjectAltName=DNS:localhost,DNS:www.wangtaoj.com,IP:127.0.0.1"
  
# 查看证书内容
openssl x509 -in wangtaoj.crt -text -noout
```

参数说明：

- `-x509`：生成自签名证书
- `-newkey rsa:4096`：生成 4096 位的 RSA 私钥
- `-sha256`：使用 SHA-256 签名算法
- `-days 3650`：证书有效期（10年）
- `-nodes`：不加密私钥（无密码）
- `-keyout server.key`：私钥输出文件
- `-out server.crt`：证书输出文件
- `-subj`：证书主题（按需修改）：
  - `C`：国家（如 CN，代表中国）
  - `ST`：省/州
  - `L`：城市
  - `O`：组织名
  - `OU`：部门
  - `CN`：主域名
- `-addext`：添加扩展（支持多域名/IP）(必须添加)：
  - `DNS:localhost`：域名
  - `IP:127.0.0.1`：IP 地址

关键说明：

1. **SAN（Subject Alternative Name）**：

   现代浏览器要求证书包含 SAN 扩展，务必在 `subjectAltName` 中添加所有需要支持的域名/IP，**否则浏览器会提示没有指定主题备用名称**

2. 上面的证书便可以通过localhost、www.wangtaoj.com、127.0.0.1访问，并且是安全的。

#### 操作系统安装该证书文件

macos需要在钥匙串访问->登录页面，将证书拖进去，然后双击证书，打开详情在信任里选择始终信任即可。

#### Spring Boot应用https配置

pem格式

```yml
server:
  port: 443
  ssl:
    certificate: classpath:certificate/wangtaoj.crt
    certificate-private-key: classpath:certificate/private.key
```

pkcs12文件

```yaml
server:
  port: 443
  ssl:
    key-store-password: 123456
    key-store-type: PKCS12
    key-store: classpath:certificate/wastonl.p12
```

