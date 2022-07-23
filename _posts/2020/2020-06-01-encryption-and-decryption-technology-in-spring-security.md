---
title: PasswordEncoder
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2020-06-01 21:33:00 +0800
categories: [Java, Spring]
tags:  [Spring Security]
math: true
mermaid: true
---

用户认证的过程会涉及到密码的校验，密码的安全性也是需要考虑的核心问题。

Spring Security 作为一个功能完备的安全性框架，除了了提供了完成认证操作的 PasswordEndocer 组件，还包一个独立完整的加密模块。
​

整个用户认证流程：在 AuthenticationProvider 中，使用  PasswordEncoder 组件验证密码的正确性，如下图说是：


## 自定义 PasswordEncoder
尽管 Spring Security 已经提供了丰富的 PasswordEncoder，可以通过实现这个接口来设计满足自身需求的任意一种密码编解码和验证机制。
​

例如，如下所示的一个 PlainTextPasswordEncoder：
```java
public class PlainTextPasswordEncoder implements PasswordEncoder {

 

   @Override
   public String encode(CharSequence rawPassword) {
       return rawPassword.toString(); 
   }


   @Override
   public boolean matches(
       CharSequence rawPassword, String encodedPassword) {
         return rawPassword.equals(encodedPassword); 
   }

}

```
PlainTextPasswordEncoder 的功能与 NoOpPasswordEncoder 类似，没有对明文进行任何处理。如果想使用某种算法集成 PasswordEncoder，就可以实现类似如下所示的 Sha512PasswordEncoder，这里使用了 SHA-512 作为加解密算法：
```java
public class Sha512PasswordEncoder implements PasswordEncoder {
    
   @Override
   public String encode(CharSequence rawPassword) {
      return hashWithSHA512(rawPassword.toString());
   }


   @Override
   public boolean matches(CharSequence rawPassword, String encodedPassword) {
      String hashedPassword = encode(rawPassword);
  	  return encodedPassword.equals(hashedPassword);
   }



   private String hashWithSHA512(String input) {
     StringBuilder result = new StringBuilder();
     try {
       MessageDigest md = MessageDigest.getInstance("SHA-512");
       byte [] digested = md.digest(input.getBytes());
       for (int i = 0; i < digested.length; i++) {
           result.append(Integer.toHexString(0xFF & digested[i]));
       } catch (NoSuchAlgorithmException e) {
           throw new RuntimeException("Bad algorithm");
     }
     return result.toString();
  }
       
}

```
上述代码中，hashWithSHA512() 方法就使用了前面提到的**单向散列加密算法**来生成消息摘要（Message Digest），其主要特点在于**单向不可逆和密文长度固定**。同时也具备“碰撞”少的优点，即明文的微小差异就会导致所生成密文完全不同。
​

SHA（Secure Hash Algorithm）以及MD5（Message Digest 5）都是常见的单向散列加密算法。
​

在 JDK 自带的 MessageDigest 类中已经包含了默认实现，我们直接调用方法即可。


## 代理式 DelegatingPasswordEncoder
在对密码进行加解密过程中，只会使用到一个 PasswordEncoder，如果这个 PasswordEncoder 不满足需求，那么就需要替换成另一个 PasswordEncoder。
​

这就引出了一个问题，Spring Security 如何优雅地应对这种变化呢?
​

虽然 DelegatingPasswordEncoder 也实现了 PasswordEncoder 接口，但事实上，它更多扮演了一种代理组件的角色，这点从命名上也可以看出来。 
​

DelegatingPasswordEncoder 将具体编码的实现根据要求代理给不同的算法，以此实现不同编码算法之间的兼容并协调变化，如下图所示：
​

# Spring Security 加密模块
​

使用 Spring Security 时，涉及用户认证的部分会用到加解密技术。就应用场景而言，加解密技术是一种通用的基础设施类技术，不仅可以用于用户认证，也可以用于其他任何涉及敏感数据处理的场景。
​

因此，Spring Security 也充分考虑到了这种需求，专门提供了一个加密模式（Spring Security Crypto Module，SSCM）。尽管 PasswordEncoder 也属于这个模块的一部分，但这个模块本身是高度独立的，可以脱离于用户认证流程来使用这个模块。
​

​

Spring Security 加密模块的核心功能有两部分。
首先就是加解密器（Encryptors），典型的使用方式如下：
```java
BytesEncryptor e = Encryptors.standard(password, salt);
```
上述方法使用了标准的 256 位 AES 算法对输入的 password 字段进行加密，返回的是一个 BytesEncryptor。
​

同时，也看到这里需要输入一个代表盐值的 salt 字段，而这个 salt 值的获取就可以用到 Spring Security 加密模块的另一个功能——键生成器（Key Generators），使用方式如下所示：
```java
String salt = KeyGenerators.string().generateKey();
```
将加解密器和键生成器结合起来，就可以实现通用的加解密机制，如下所示：
```java
String salt = KeyGenerators.string().generateKey(); 
String password = "secret"; 
String valueToEncrypt = "HELLO"; 
BytesEncryptor e = Encryptors.standard(password, salt); 

byte [] encrypted = e.encrypt(valueToEncrypt.getBytes()); 
byte [] decrypted = e.decrypt(encrypted);

```
在日常开发过程中，可以根据需要调整上述代码并嵌入到自己的系统中。
​

​

对于一个 Web 应用程序来说，一旦需要实现用户认证，势必涉及用户密码等敏感信息的加密。为此 Spring Security 提供 PasswordEncoder 组件（内置一批即插即用的 PasswordEncoder）对密码进行加解密，并通过代理机制完成了各个组件的版本兼容和统一管理。这种思想值得学习和借鉴。
