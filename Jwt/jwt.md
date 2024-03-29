### 简介

JWT(JSON Web Token)是一种去中心化的web认证方案，信息存储在客户端。

### 数据结构

JWT通常由3部分组成，Header、Payload、Signature。每个部分都是用Base64Url编码后的字符串，每个部分之间由点分割。形如

```txt
Header.Payload.Signature
```

注: Base64Url是Base64的一个变种，主要是将Base64编码之后的"+"使用"-"替换，"/"使用"_"替换。因为字符+和/在URL中不是安全的字符。

#### Header

Header 部分是一个 JSON 对象，描述 JWT 的元数据，通常是下面的样子。

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

其中alg表示签名算法，typ属性表示这个令牌（token）的类型（type），JWT 令牌统一写为JWT。

#### Payload

Payload 部分也是一个 JSON 对象，用来存放实际需要传递的数据。JWT 规定了7个官方字段，供选用。

- iss (issuer)：签发人
- exp (expiration time)：过期时间
- sub (subject)：主题
- aud (audience)：受众
- nbf (Not Before)：生效时间
- iat (Issued At)：签发时间
- jti (JWT ID)：编号

除了官方字段，你还可以在这个部分定义私有字段，下面就是一个例子。

```json
{
    "loginName": "xxxx",
    "nickname": "xxxxx",
    "sex": "男"
}
```

#### Signature

Signature是对前面两部分的一个签名，防止数据被篡改。比如使用RS256签名算法，则可以使用以下公式可以得到签名部分。

```tex
Signature=rsa(sha256(base64EncodeUrl(Header) + "." + base64EncodeUrl(Payload)), privateKey)
```

即使用base64Url编码Header、Payload，然后再使用sha256摘要算法取得hash值，最后再用rsa非对称算法使用私钥进行加密。

**注意: 发送给客户端的JWT字符串的每一部分都是经过Base64URL编码后的字符串，然后使用点连接** 。

### 验证JWT是否合法

既然知道了怎么签名的，那么解签就容易了

拿到JWT字符串后，截取前面的header+payload，这里已经是经过base64Url编码之后的结果了，最后一部分则是签名部分。

用rsa算法根据公钥解密签名，便能拿到sha256算法的hash值了，于是用这个值再和header+payload经过sha256 hash后的值比较即可。

注：基于RS256签名算法，签名需要使用私钥，因为token是服务端生成的，第三方使用公钥解签，第三方可以有很多。私钥保密，公钥公开。比如统一认证中心生成JWT字符串，应用a，应用b都能使用公开的公钥来验证签名是否合法的。

### 签名与解签代码演示

生成合法的RSA配对秘钥用于测试

```java
@Test
public void generateKey() throws Exception {
    KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
    // 设置密钥长度，通常为 2048 位
    keyPairGenerator.initialize(2048);
    KeyPair keyPair = keyPairGenerator.generateKeyPair();
    PublicKey publicKey = keyPair.getPublic();
    PrivateKey privateKey = keyPair.getPrivate();

    // 使用base64编码, 使用方使用base64解码得到字节数组即可
    System.out.println("PublicKey: " + Base64.getEncoder().encodeToString(publicKey.getEncoded()));
    System.out.println("PrivateKey: " + Base64.getEncoder().encodeToString(privateKey.getEncoded()));
}
```



签名与验签

```java
public class SignatureAlgTest {

    /**
     * 注意公钥和私钥必须是合法的RSA配对秘钥
     */
    @Test
    public void testSignAndVerify() throws Exception{
        final String publicKey ="MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvzFky4s0gcmGytnaIS4JT/pcuW6Yn565mFjT3V1Qo5AMDMSvL5lp7fixGqtUQWDm1az+Vu6QMbLJAR6HyeNMa9EfkhUJQPFFNg29ydDXqzJjhdfdo9O78V20Pwu7ud7MRCq05COU6FMQ+sZmBrylokMyB5YDyBEB/bMo5pcEaeJAq9cd3ORbLBWKJz8NU6nPSkSJEjX2DGRekH/+lQazmpB+kUg8b7rw6pfYwLcSsWpvcgnHWeExuS7vGLQu2cT3SlAfUu9dp+o5ECQX8OmM7YzyMXuTd8D4ijSV8ZPfLAuktuSjMiX0rTHwxBWSUm6LIjNiF5jW9lWUU92mVPxpvwIDAQAB";
        final String privateKey = "MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC/MWTLizSByYbK2dohLglP+ly5bpifnrmYWNPdXVCjkAwMxK8vmWnt+LEaq1RBYObVrP5W7pAxsskBHofJ40xr0R+SFQlA8UU2Db3J0NerMmOF192j07vxXbQ/C7u53sxEKrTkI5ToUxD6xmYGvKWiQzIHlgPIEQH9syjmlwRp4kCr1x3c5FssFYonPw1Tqc9KRIkSNfYMZF6Qf/6VBrOakH6RSDxvuvDql9jAtxKxam9yCcdZ4TG5Lu8YtC7ZxPdKUB9S712n6jkQJBfw6YztjPIxe5N3wPiKNJXxk98sC6S25KMyJfStMfDEFZJSbosiM2IXmNb2VZRT3aZU/Gm/AgMBAAECggEALHel/Es3npoK/iH2ADKPWukdaMlmuPU3ME40lG8wIqKNkuip4BW70+u78Tp44a3Sck8GZpycr9pnsplxtoxliUv9nkHDQbX7xWsjwY0PpBMXn5kJxSEpPKVxFxq5Ai1l79LI+Kin6PLs545+S0HT+i3LtIT5Ay6lemaRdDQahC+CmSlHCTcaz38RnjFLntYGPQcjWZUqN5ROgizwCDmt/ZE2xM9LH9Wrp1lTl5D7vkLmNTIAJ+Bm3ANat0OjhMpQ1Dmc82NYT8BNzWRx25uhsRHcS7+NcVQk35WMiZXRL3+t18saNSQOYXxmlO5qrCIOjyUGi9WNYTrGh8KlgchhSQKBgQDn/XFW+HnknJJIRh3GCValymWgnXbKzGlRuqKIoCjoiIjOBQaBshQ+j3IdwortxBmgHbx4uh2z2xaPGynSwJXVKoHoksY6zlJ4Bb5TaQgxapEcqOlHIfUkZfE6O5W/MHP4ACvxl0lIod4B45YCLRFvDoLneIIBXhvL7ioUt/pxdwKBgQDS+wkwvEGd7T10baxYbXU2iBo43hPHIUU1JwdNaQjrZ8QMyVKm0/mwqvVPn5BqGiO3Jq7QV+vjcAh1LOwJGzo9xLOD2UXaPAtDA9OUHAfL6Od45eVweM3h9/ecL2pmEElPqSS8puzFE6MA5qU1KEyL4x/x9jIQaBAihx7tYheb+QKBgB2fDsm8EFRQaZ0w1rxilN22aiOH95MNZqU432fyi0alqFIl8h69TjhuuHN0U6joUR1Qrq/7k69TWh4LqdtvG7KMKuo3U3hOv9jzYsnjr1gf80dlieO7QkHTgmmdEhHHbgdMfk/qsUDE6kPze0Pr3T4A7FYB3RevnHz9fAIJO8EhAoGARgxBBeRLKOL+l2xeX1GgLAXOJvlcua2LK9WUcBgidP4TsmcZQPh6GzT3k4MX0JJzLzjxq4y1beLhe/35NCDNGnr3WxxFO+rZlltr4O3ZjNL8H0C9B7WkLZVFqZ54hgB8Rq2S2+vUCq61XPQ2/8osd/llvtEN2DKkwMH5+7iovAkCgYEAyge6ONdXEhkTCsIbYbOgzL5CUBAQERt2+KGHZC9AeVQwl9xZUznDqKvdKz3A5io7PybSugMLNhC+5ZERFbXf7itanR+iXD68k0ySQcDPBVysQsIrOWqaIeGwM7vIn45BJLdW/YV/wHX36Bp0Sp+CNxJLosf2VzUDJuFyPh0wIhs=";

        // 需要签名的内容
        final String text = "hello world";

        // 最终的签名字符串(JWT的signature部分), 使用base64URL编码
        String signStr = sign(text, privateKey);

        // 验签
        boolean verify = verifySign(text, signStr, publicKey);
        Assertions.assertTrue(verify);

        // 修改签名字符串(不能直接修改字符串, 自己乱修改的可能不符合base64URL编码规范)
        byte[] signBytes = Base64.getUrlDecoder().decode(signStr);
        byte[] signBytesUpdate = Arrays.copyOf(signBytes, signBytes.length);
        signBytesUpdate[0] = 97;
        String signStrUpdate = Base64.getUrlEncoder().encodeToString(signBytesUpdate);

        // 再次验签, 由于篡改了内容, 因此验签失败
        verify = verifySign(text, signStrUpdate, publicKey);
        Assertions.assertFalse(verify);
    }

    /**
     * 签名
     */
    private String sign(String text, String privateKey) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        // 使用私钥签名
        signature.initSign(privateKey(privateKey));
        signature.update(text.getBytes(StandardCharsets.UTF_8));

        byte[] signBytes = signature.sign();
        // 最终的签名字符串(JWT的signature部分), 使用base64URL编码
        return Base64.getUrlEncoder().encodeToString(signBytes);
    }

    /**
     * 验签
     */
    private boolean verifySign(String text, String signStr, String publicKey) throws Exception {
        // 解签验证
        Signature signature = Signature.getInstance("SHA256withRSA");
        // 使用公钥验签
        signature.initVerify(publicKey(publicKey));
        signature.update(text.getBytes(StandardCharsets.UTF_8));
        return signature.verify(Base64.getUrlDecoder().decode(signStr));
    }
    
     /**
     * 使用给定字符串构建公钥实例
     * @param key base64编码的字符串
     */
    private PublicKey publicKey(String key) throws Exception{
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        byte[] keyBytes = Base64.getDecoder().decode(key);
        // RSA公钥需要使用X509EncodedKeySpec
        return keyFactory.generatePublic(new X509EncodedKeySpec(keyBytes));
    }

    /**
     * 使用给定字符串构建私钥实例
     * @param key base64编码的字符串
     */
    private PrivateKey privateKey(String key) throws Exception{
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        byte[] keyBytes = Base64.getDecoder().decode(key);
        // RSA私钥需要使用PKCS8EncodedKeySpec
        return keyFactory.generatePrivate(new PKCS8EncodedKeySpec(keyBytes));
    }
}
```

### JWT API使用

使用jjwt库(Java JWT)

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.12.5</version>
</dependency>
```

```java
public class JwtTest {

    @Test
    public void testCreateAndParse() {
        KeyPair keyPair = Jwts.SIG.RS256.keyPair().build();
        PrivateKey privateKey = keyPair.getPrivate();
        PublicKey publicKey = keyPair.getPublic();

        Map<String, Object> customPayload = new HashMap<>();
        customPayload.put("loginName", "wastonl");
        customPayload.put("sex", "男");
        customPayload.put("age", 18);

        long currentTime = System.currentTimeMillis();
        String jwt = Jwts.builder()
                .claims(customPayload)
                // 签发时间
                .issuedAt(new Date(currentTime))
                // 1小时后过期
                .expiration(new Date(currentTime + 1000 * 60 * 60))
                .signWith(privateKey, Jwts.SIG.RS256)
                .compact();
        System.out.println(jwt);

        Jws<Claims> claimsJws = Jwts.parser().verifyWith(publicKey)
                .build()
                .parseSignedClaims(jwt);
        System.out.println(claimsJws.getPayload());
    }
}
```

