---
layout: post
title: 'Java使用OpenSSL生成RSA进行加密和解密'
date: 2018-06-04
author: 李新
tags: OpenSSL
---

### (0). 参考博客
> 本文内容参考于(https://www.cnblogs.com/linjiqin/p/11133435.html)  

### (1). 查看OpenSSL版本
```
lixin-macbook:~ lixin$ openssl version -a
LibreSSL 2.8.3
```
### (2). 创建私钥
```
# 创建证书目录
lixin-macbook:~ lixin$ cd ~/Desktop/
lixin-macbook:Desktop lixin$ mkdir rsa
lixin-macbook:Desktop lixin$ cd rsa/



# 生成私钥
# -out rsa_private_key.pem    :  将生成的私钥保存至指定的文件中.
# numbits                     : 指定生成私钥的大小,默认是2048.
lixin-macbook:rsa lixin$ openssl genrsa -out rsa_private_key.pem  2048
Generating RSA private key, 2048 bit long modulus
.................................+++
.........................................................+++
e is 65537 (0x10001)
```
### (3). 根据私钥提取公钥
```
# -in rsa_private_key.pem  : 指明公钥文件
# -pubout                  : 根据私钥提取出公钥
# -out rsa_public_key.pem  : 提取出的公钥保存位置
lixin-macbook:rsa lixin$ openssl rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem
writing RSA key
```
### (4). 对私钥进行PKCS#8编码
```
# 
# 指明输入私钥文件为rsa_private_key.pem,输出私钥文件为pkcs8_rsa_private_key.pem,不采用任何二次加密(-nocrypt)
lixin-macbook:rsa lixin$ openssl pkcs8 -topk8 -in rsa_private_key.pem -out pkcs8_rsa_private_key.pem -nocrypt

lixin-macbook:rsa lixin$ ll
-rw-r--r--   1 lixin  staff  1704  2 11 21:10 pkcs8_rsa_private_key.pem
-rw-r--r--   1 lixin  staff  1679  2 11 21:01 rsa_private_key.pem
-rw-r--r--   1 lixin  staff   451  2 11 21:06 rsa_public_key.pem
```
### (5). 添加POM依赖
```
<dependency>
	<groupId>commons-codec</groupId>
	<artifactId>commons-codec</artifactId>
	<version>1.11</version>
</dependency>
```
### (6). RSAEncrypt
```
package help.lixin;

import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;
import java.security.InvalidKeyException;
import java.security.KeyFactory;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.security.interfaces.RSAPrivateKey;
import java.security.interfaces.RSAPublicKey;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.X509EncodedKeySpec;

import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;

import org.apache.commons.codec.binary.Base64;

import sun.misc.BASE64Decoder;

public class RSAEncrypt {
	/**
	 * 字节数据转字符串专用集合
	 */
	private static final char[] HEX_CHAR = { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e',
			'f' };

	private static final String PRIVATE_KEY = "/pkcs8_rsa_private_key.pem";

	private static final String PUBLIC_KEY = "/rsa_public_key.pem";

	/**
	 * 随机生成密钥对
	 */
	public static void genKeyPair(String filePath) {
		// KeyPairGenerator类用于生成公钥和私钥对，基于RSA算法生成对象
		KeyPairGenerator keyPairGen = null;
		try {
			keyPairGen = KeyPairGenerator.getInstance("RSA");
		} catch (NoSuchAlgorithmException e) {
			e.printStackTrace();
		}
		// 初始化密钥对生成器，密钥大小为96-1024位
		keyPairGen.initialize(1024, new SecureRandom());
		// 生成一个密钥对，保存在keyPair中
		KeyPair keyPair = keyPairGen.generateKeyPair();
		// 得到私钥
		RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
		// 得到公钥
		RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
		try {
			// 得到公钥字符串
			Base64 base64 = new Base64();
			String publicKeyString = new String(base64.encode(publicKey.getEncoded()));
			// 得到私钥字符串
			String privateKeyString = new String(base64.encode(privateKey.getEncoded()));
			// 将密钥对写入到文件
			FileWriter pubfw = new FileWriter(filePath + PUBLIC_KEY);
			FileWriter prifw = new FileWriter(filePath + PRIVATE_KEY);
			BufferedWriter pubbw = new BufferedWriter(pubfw);
			BufferedWriter pribw = new BufferedWriter(prifw);
			pubbw.write(publicKeyString);
			pribw.write(privateKeyString);
			pubbw.flush();
			pubbw.close();
			pubfw.close();
			pribw.flush();
			pribw.close();
			prifw.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 从文件中输入流中加载公钥
	 *
	 * @param path 公钥输入流
	 * @throws Exception 加载公钥时产生的异常
	 */
	public static String loadPublicKeyByFile(String path) throws Exception {
		try {
			BufferedReader br = new BufferedReader(new FileReader(path + PUBLIC_KEY));
			String readLine = null;
			StringBuilder sb = new StringBuilder();
			while ((readLine = br.readLine()) != null) {
				if (readLine.charAt(0) == '-') {
					continue;
				} else {
					sb.append(readLine);
					sb.append('\r');
				}
			}
			br.close();
			return sb.toString();
		} catch (IOException e) {
			throw new Exception("公钥数据流读取错误");
		} catch (NullPointerException e) {
			throw new Exception("公钥输入流为空");
		}
	}

	/**
	 * 从字符串中加载公钥
	 *
	 * @param publicKeyStr 公钥数据字符串
	 * @throws Exception 加载公钥时产生的异常
	 */
	public static RSAPublicKey loadPublicKeyByStr(String publicKeyStr) throws Exception {
		try {
			BASE64Decoder base64 = new BASE64Decoder();
			byte[] buffer = base64.decodeBuffer(publicKeyStr);
			KeyFactory keyFactory = KeyFactory.getInstance("RSA");
			X509EncodedKeySpec keySpec = new X509EncodedKeySpec(buffer);
			return (RSAPublicKey) keyFactory.generatePublic(keySpec);
		} catch (NoSuchAlgorithmException e) {
			throw new Exception("无此算法");
		} catch (InvalidKeySpecException e) {
			throw new Exception("公钥非法");
		} catch (NullPointerException e) {
			throw new Exception("公钥数据为空");
		}
	}

	/**
	 * 从文件中加载私钥
	 *
	 * @param path 私钥文件名
	 * @return 是否成功
	 * @throws Exception
	 */
	public static String loadPrivateKeyByFile(String path) throws Exception {
		try {
			BufferedReader br = new BufferedReader(new FileReader(path + PRIVATE_KEY));
			String readLine = null;
			StringBuilder sb = new StringBuilder();
			while ((readLine = br.readLine()) != null) {
				if (readLine.charAt(0) == '-') {
					continue;
				} else {
					sb.append(readLine);
					sb.append('\r');
				}
			}
			br.close();
			return sb.toString();
		} catch (IOException e) {
			throw new Exception("私钥数据读取错误");
		} catch (NullPointerException e) {
			throw new Exception("私钥输入流为空");
		}
	}

	public static RSAPrivateKey loadPrivateKeyByStr(String privateKeyStr) throws Exception {
		try {
			BASE64Decoder base64Decoder = new BASE64Decoder();
			byte[] buffer = base64Decoder.decodeBuffer(privateKeyStr);
			PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(buffer);
			KeyFactory keyFactory = KeyFactory.getInstance("RSA");
			return (RSAPrivateKey) keyFactory.generatePrivate(keySpec);
		} catch (NoSuchAlgorithmException e) {
			throw new Exception("无此算法");
		} catch (InvalidKeySpecException e) {
			throw new Exception("私钥非法");
		} catch (NullPointerException e) {
			throw new Exception("私钥数据为空");
		}
	}

	/**
	 * 公钥加密过程
	 *
	 * @param publicKey     公钥
	 * @param plainTextData 明文数据
	 * @return
	 * @throws Exception 加密过程中的异常信息
	 */
	public static byte[] encrypt(RSAPublicKey publicKey, byte[] plainTextData) throws Exception {
		if (publicKey == null) {
			throw new Exception("加密公钥为空, 请设置");
		}
		Cipher cipher = null;
		try {
			// 使用默认RSA
			cipher = Cipher.getInstance("RSA");
			// cipher= Cipher.getInstance("RSA", new BouncyCastleProvider());
			cipher.init(Cipher.ENCRYPT_MODE, publicKey);
			byte[] output = cipher.doFinal(plainTextData);
			return output;
		} catch (NoSuchAlgorithmException e) {
			throw new Exception("无此加密算法");
		} catch (NoSuchPaddingException e) {
			e.printStackTrace();
			return null;
		} catch (InvalidKeyException e) {
			throw new Exception("加密公钥非法,请检查");
		} catch (IllegalBlockSizeException e) {
			throw new Exception("明文长度非法");
		} catch (BadPaddingException e) {
			throw new Exception("明文数据已损坏");
		}
	}

	/**
	 * 私钥加密过程
	 *
	 * @param privateKey    私钥
	 * @param plainTextData 明文数据
	 * @return
	 * @throws Exception 加密过程中的异常信息
	 */
	public static byte[] encrypt(RSAPrivateKey privateKey, byte[] plainTextData) throws Exception {
		if (privateKey == null) {
			throw new Exception("加密私钥为空, 请设置");
		}
		Cipher cipher = null;
		try {
			// 使用默认RSA
			cipher = Cipher.getInstance("RSA");
			cipher.init(Cipher.ENCRYPT_MODE, privateKey);
			byte[] output = cipher.doFinal(plainTextData);
			return output;
		} catch (NoSuchAlgorithmException e) {
			throw new Exception("无此加密算法");
		} catch (NoSuchPaddingException e) {
			e.printStackTrace();
			return null;
		} catch (InvalidKeyException e) {
			throw new Exception("加密私钥非法,请检查");
		} catch (IllegalBlockSizeException e) {
			throw new Exception("明文长度非法");
		} catch (BadPaddingException e) {
			throw new Exception("明文数据已损坏");
		}
	}

	/**
	 * 私钥解密过程
	 *
	 * @param privateKey 私钥
	 * @param cipherData 密文数据
	 * @return 明文
	 * @throws Exception 解密过程中的异常信息
	 */
	public static byte[] decrypt(RSAPrivateKey privateKey, byte[] cipherData) throws Exception {
		if (privateKey == null) {
			throw new Exception("解密私钥为空, 请设置");
		}
		Cipher cipher = null;
		try {
			// 使用默认RSA
			cipher = Cipher.getInstance("RSA");
			// cipher= Cipher.getInstance("RSA", new BouncyCastleProvider());
			cipher.init(Cipher.DECRYPT_MODE, privateKey);
			byte[] output = cipher.doFinal(cipherData);
			return output;
		} catch (NoSuchAlgorithmException e) {
			throw new Exception("无此解密算法");
		} catch (NoSuchPaddingException e) {
			e.printStackTrace();
			return null;
		} catch (InvalidKeyException e) {
			throw new Exception("解密私钥非法,请检查");
		} catch (IllegalBlockSizeException e) {
			throw new Exception("密文长度非法");
		} catch (BadPaddingException e) {
			throw new Exception("密文数据已损坏");
		}
	}

	/**
	 * 公钥解密过程
	 *
	 * @param publicKey  公钥
	 * @param cipherData 密文数据
	 * @return 明文
	 * @throws Exception 解密过程中的异常信息
	 */
	public static byte[] decrypt(RSAPublicKey publicKey, byte[] cipherData) throws Exception {
		if (publicKey == null) {
			throw new Exception("解密公钥为空, 请设置");
		}
		Cipher cipher = null;
		try {
			// 使用默认RSA
			cipher = Cipher.getInstance("RSA");
			// cipher= Cipher.getInstance("RSA", new BouncyCastleProvider());
			cipher.init(Cipher.DECRYPT_MODE, publicKey);
			byte[] output = cipher.doFinal(cipherData);
			return output;
		} catch (NoSuchAlgorithmException e) {
			throw new Exception("无此解密算法");
		} catch (NoSuchPaddingException e) {
			e.printStackTrace();
			return null;
		} catch (InvalidKeyException e) {
			throw new Exception("解密公钥非法,请检查");
		} catch (IllegalBlockSizeException e) {
			throw new Exception("密文长度非法");
		} catch (BadPaddingException e) {
			throw new Exception("密文数据已损坏");
		}
	}

	/**
	 * 字节数据转十六进制字符串
	 *
	 * @param data 输入数据
	 * @return 十六进制内容
	 */
	public static String byteArrayToString(byte[] data) {
		StringBuilder stringBuilder = new StringBuilder();
		for (int i = 0; i < data.length; i++) {
			// 取出字节的高四位 作为索引得到相应的十六进制标识符 注意无符号右移
			stringBuilder.append(HEX_CHAR[(data[i] & 0xf0) >>> 4]);
			// 取出字节的低四位 作为索引得到相应的十六进制标识符
			stringBuilder.append(HEX_CHAR[(data[i] & 0x0f)]);
			if (i < data.length - 1) {
				stringBuilder.append(' ');
			}
		}
		return stringBuilder.toString();
	}
}
```
### (7). RSASignature
```
package help.lixin;

import java.security.PrivateKey;
import java.security.PublicKey;

public class RSASignature {

	/**
	 * 签名算法
	 */
	public static final String SIGN_ALGORITHMS = "hwU-gARp25KqMFVxw-9J3eCfRNXSN9QM0,ymavRvG0ByPpWEx-IJ{I0DE3w6LMP0fUwKDmSevTpFf=1Q}-h8B,14UGHs4-{flavC";

	/**
	 * RSA签名
	 *
	 * @param content    待签名数据
	 * @param privateKey 商户私钥
	 * @param encode     字符集编码
	 * @return 签名值
	 */
	public static String sign(String content, PrivateKey privateKey, String encode) {
		try {
			java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);
			signature.initSign(privateKey);
			signature.update(content.getBytes(encode));
			byte[] signed = signature.sign();
			return byte2Hex(signed);
		} catch (Exception e) {
			e.printStackTrace();
		}

		return null;
	}

	public static String sign(String content, PrivateKey privateKey) {
		try {
			java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);
			signature.initSign(privateKey);
			signature.update(content.getBytes());
			byte[] signed = signature.sign();
			return byte2Hex(signed);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	/**
	 * 将byte[] 转换成字符串
	 */
	public static String byte2Hex(byte[] srcBytes) {
		StringBuilder hexRetSB = new StringBuilder();
		for (byte b : srcBytes) {
			String hexString = Integer.toHexString(0x00ff & b);
			hexRetSB.append(hexString.length() == 1 ? 0 : "").append(hexString);
		}
		return hexRetSB.toString();
	}

	/**
	 * RSA验签名检查
	 *
	 * @param content   待签名数据
	 * @param sign      签名值
	 * @param publicKey 分配给开发商公钥
	 * @param encode    字符集编码
	 * @return 布尔值
	 */
	public static boolean doCheck(String content, String sign, PublicKey publicKey, String encode) {
		try {
			java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);

			signature.initVerify(publicKey);
			signature.update(content.getBytes(encode));

			boolean bverify = signature.verify(hex2Bytes(sign));
			return bverify;

		} catch (Exception e) {
			e.printStackTrace();
		}

		return false;
	}

	public static boolean doCheck(String content, String sign, PublicKey publicKey) {
		try {
			java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);
			signature.initVerify(publicKey);
			signature.update(content.getBytes());
			boolean bverify = signature.verify(hex2Bytes(sign));
			return bverify;
		} catch (Exception e) {
			e.printStackTrace();
		}

		return false;
	}

	/**
	 * 将16进制字符串转为转换成字符串
	 */
	public static byte[] hex2Bytes(String source) {
		byte[] sourceBytes = new byte[source.length() / 2];
		for (int i = 0; i < sourceBytes.length; i++) {
			sourceBytes[i] = (byte) Integer.parseInt(source.substring(i * 2, i * 2 + 2), 16);
		}
		return sourceBytes;
	}
}
```
### (8). RSASignature
```
package help.lixin;

import java.security.PrivateKey;
import java.security.PublicKey;

public class RSASignature {

	/**
	 * 签名算法
	 */
	public static final String SIGN_ALGORITHMS = "hwU-gARp25KqMFVxw-9J3eCfRNXSN9QM0,ymavRvG0ByPpWEx-IJ{I0DE3w6LMP0fUwKDmSevTpFf=1Q}-h8B,14UGHs4-{flavD";

	/**
	 * RSA签名
	 *
	 * @param content    待签名数据
	 * @param privateKey 私钥
	 * @param encode     字符集编码
	 * @return 签名值
	 */
	public static String sign(String content, PrivateKey privateKey, String encode) {
		try {
			java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);
			signature.initSign(privateKey);
			signature.update(content.getBytes(encode));
			byte[] signed = signature.sign();
			return byte2Hex(signed);
		} catch (Exception e) {
			e.printStackTrace();
		}

		return null;
	}

	public static String sign(String content, PrivateKey privateKey) {
		try {
			java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);
			signature.initSign(privateKey);
			signature.update(content.getBytes());
			byte[] signed = signature.sign();
			return byte2Hex(signed);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	/**
	 * 将byte[] 转换成字符串
	 */
	public static String byte2Hex(byte[] srcBytes) {
		StringBuilder hexRetSB = new StringBuilder();
		for (byte b : srcBytes) {
			String hexString = Integer.toHexString(0x00ff & b);
			hexRetSB.append(hexString.length() == 1 ? 0 : "").append(hexString);
		}
		return hexRetSB.toString();
	}

	/**
	 * RSA验签名检查
	 *
	 * @param content   待签名数据
	 * @param sign      签名值
	 * @param publicKey 分配给公钥
	 * @param encode    字符集编码
	 * @return 布尔值
	 */
	public static boolean doCheck(String content, String sign, PublicKey publicKey, String encode) {
		try {
			java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);

			signature.initVerify(publicKey);
			signature.update(content.getBytes(encode));

			boolean bverify = signature.verify(hex2Bytes(sign));
			return bverify;

		} catch (Exception e) {
			e.printStackTrace();
		}

		return false;
	}

	public static boolean doCheck(String content, String sign, PublicKey publicKey) {
		try {
			java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);
			signature.initVerify(publicKey);
			signature.update(content.getBytes());
			boolean bverify = signature.verify(hex2Bytes(sign));
			return bverify;
		} catch (Exception e) {
			e.printStackTrace();
		}

		return false;
	}

	/**
	 * 将16进制字符串转为转换成字符串
	 */
	public static byte[] hex2Bytes(String source) {
		byte[] sourceBytes = new byte[source.length() / 2];
		for (int i = 0; i < sourceBytes.length; i++) {
			sourceBytes[i] = (byte) Integer.parseInt(source.substring(i * 2, i * 2 + 2), 16);
		}
		return sourceBytes;
	}
}
```
### (9). Test
```
package help.lixin;

import org.apache.commons.codec.binary.Base64;

public class Test {
	public static void main(String[] args) throws Exception {
		String publicPath = "/Users/lixin/Desktop/rsa"; // 公匙存放位置
		String privatePath = "/Users/lixin/Desktop/rsa"; // 私匙存放位置
		Base64 base64 = new Base64();

		System.out.println("--------------公钥加密私钥解密过程-------------------");
		String signKey = "明文内容......";
		// 公钥对明文加密过程
		byte[] cipherData = RSAEncrypt
				.encrypt(RSAEncrypt.loadPublicKeyByStr(RSAEncrypt.loadPublicKeyByFile(publicPath)), signKey.getBytes());
		String cipher = new String(base64.encode(cipherData));

		// 私钥对密文解密过程
		byte[] res = RSAEncrypt.decrypt(RSAEncrypt.loadPrivateKeyByStr(RSAEncrypt.loadPrivateKeyByFile(privatePath)),
				base64.decode(cipher));
		String restr = new String(res);
		System.out.println("原文：" + signKey);
		System.out.println("加密：" + cipher);
		System.out.println("解密：" + restr);
		System.out.println();

		System.out.println("--------------私钥加密公钥解密过程-------------------");
		// 私钥加密过程
		cipherData = RSAEncrypt.encrypt(RSAEncrypt.loadPrivateKeyByStr(RSAEncrypt.loadPrivateKeyByFile(privatePath)),
				signKey.getBytes());
		cipher = new String(base64.encode(cipherData));
		// 公钥解密过程
		res = RSAEncrypt.decrypt(RSAEncrypt.loadPublicKeyByStr(RSAEncrypt.loadPublicKeyByFile(publicPath)),
				base64.decode(cipher));
		restr = new String(res);
		System.out.println("原文：" + signKey);
		System.out.println("加密：" + cipher);
		System.out.println("解密：" + restr);
		System.out.println();
	}
}
```
### (10). 查看结果
```
--------------公钥加密私钥解密过程-------------------
原文：明文内容......
加密：LrqSKvvQv2uyAqf6m7Q+M0rRv2uRfYntTpd030+6tuzrVBjDZSS0tQFASvhWlhOyJ2R6TJx0cBA1Hgvc7tB/O2B128RcEodEG5KVg4Qyajb+KhGqdeLXdbeA2XxOPVUa53qCVFsIHj9PBHXYTidSAmLWY4uYN7wZXvbR0j8mHLK3DUkrqlMnug9Z32HLvQmSjvHUJ9cv/ZcBkBr8nFlVmd1rK1vBFprvVYg2RwDJUTqz2cA4O2KoyL5Ji26OnX+NSRYUnVPSdA/3LCcDxA2xxCBEMODbFi2jbRPioGH7obLd90eF8Kk9h/f3bUk2G7z8/8Fg+B94wW5aS3qnGeMdyw==
解密：明文内容......

--------------私钥加密公钥解密过程-------------------
原文：明文内容......
加密：Dp8C8n5GYNDeSsUf1SD9qjczD6pB3jKQdRnmbLZk8SqBpjyxeLWcMWDf1zfNIeyP58FOrCh/OZdnwq1bKVDLC4ubqWnwRVqqcao6mrXhcyd8DpgcHb7yKqs57FH92hoKG3wAiJChCdBAMM6gzD7f1J8xxkTPM2LXkvH9iiopAVO7T26LTgt8XiHiUP4qwCn2UK8R6DBnRLwaKC7nNNeAcn78HW6q22QvFzxDJyJBprw9+tGfuNbbO1wxMjv/DwGDG1ZKw+uR3GlsCqay5hf9cpIZ4+BsYgTg5or4aIGCIFUgt55Nw6mSzYwZAQoI2n6sOGtDK8d7bswd5jgA09RURw==
解密：明文内容......
```