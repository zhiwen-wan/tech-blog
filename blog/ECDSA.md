# ECDSA Signature
When I worked on a project, we need send signed data from server to client with ECDSA (Elliptic Curve digital Signature Algorithm) signagure. It 
offers a variant of the digital Signature Algorithm (DSA) which uses elliptic curve cryptography, please see [wiki](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) for the algorithm. In this blog, we will talk about how to generate private key and public key with OPenSSL, and how to sign data and validate signed data.

## Generate EC(Elliptic Curve) keys
OpenSSL is great tool for generating keys suitable for EC algorithms. Follow [instruction](https://tecadmin.net/install-openssl-on-windows/) to install OpenSSL on windows. 

There are many kind of EC keys in the document, our project used prime256v1 key which is X9.62/SECG curve over a 256 bit prime field. And use these commands to generate private / public keys with pem format which is encoded with base64. If you use different EC keys, ensure the name matches your keys. 

First, generate the private key with bellowing command:

```python
PS E:\certificate>  openssl ecparam -genkey -name prime256v1 -out ecprivate.pem<br/>

	-----BEGIN EC PARAMETERS-----
	BggqhkjOPQMBBw==
	-----END EC PARAMETERS-----
	-----BEGIN EC PRIVATE KEY-----
	MHcCAQEEILTA2fzecOwG+tfR3e07yT0ihyl5q9NJO7pEpVzQfSgGoAoGCCqGSM49
	AwEHoUQDQgAECW+SD9PJxK1m94d/kRqPxb9hZP7005La42dG2VmQlq3uClLG2x+x
	eVCbgjV3i0wW0L0qaXI0z97+G7VfemZyKg==
	-----END EC PRIVATE KEY-----
```

Then, get the corresponding public key: 
```python
PS E:\certificate>  openssl ec -in ecprivate.pem -pubout -out ecpublic.pem
read EC key
writing EC key

	-----BEGIN PUBLIC KEY-----
	MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAECW+SD9PJxK1m94d/kRqPxb9h
	05La42dG2VmQlq3uClLG2x+xeVCbgjV3i0wW0L0qaXI0z97+G7VfemZyKg==
	-----END PUBLIC KEY-----
	LMGoMeoTTPV3vFLWOj1vVqtIYIOGhPzRr5FodQ==
	-----END PUBLIC KEY-----
```

If need using bytes, you can covert string into bytes:

```python
PS E:\certificate> openssl ec -in .\ecprivate.pem -text -noout
read EC key
Private-Key: (256 bit)
	priv:
	    b4:c0:d9:fc:de:70:ec:06:fa:d7:d1:dd:ed:3b:c9:
	    3d:22:87:29:79:ab:d3:49:3b:ba:44:a5:5c:d0:7d:
	    28:06
	pub: 
	    04:09:6f:92:0f:d3:c9:c4:ad:66:f7:87:7f:91:1a:
	    8f:c5:bf:61:64:fe:f4:d3:92:da:e3:67:46:d9:59:
	    90:96:ad:ee:0a:52:c6:db:1f:b1:79:50:9b:82:35:
	    77:8b:4c:16:d0:bd:2a:69:72:34:cf:de:fe:1b:b5:
	    5f:7a:66:72:2a
	ASN1 OID: prime256v1
	NIST CURVE: P-256
```

Public key is 64-byte with 0x04 header and private key is 32-byte. If you are interested in the key format,[this](https://davidederosa.com/basic-blockchain-programming/elliptic-curve-keys/) is good resource. I will give more detail about DER format in the end of blog.

## Sign and verify the file with OpenSSL 
OpenSSL provides commands to sign the file with private key and verify the signature with public key.

Sign the file with private key to DER format signature:

```python
PS E:\certificate> openssl dgst -sha256 -sign ecpriv.pem json.txt > sign.bin   
```

Verify the signature with public key:

```python
PS E:\certificate> openssl dgst -sha256 -verify ecpub.pem -signature sign.bin < json.txt
Verified OK
```
 
## Programnal Signing and verification
It is not feasible to implement the algorithm. We will use existing lib to complete signing data and verifying signature. In this blog, I will show you how to sign data and verify signature in Node JS and C#.

### Node JS
Crypto package uses to create signature with different crypto algorithm in Node JS. [This document](https://nodejs.org/api/crypto.html) lists all functions that is provided.

Look the bellowing example. There are two parts, one is for signing and the other is for verifcation. We need private key and hashed data for signing, crypto.createSign function creates signed data. With public key and hashed data, signagure can be validated by using crypto.createVerify function.

```javascript
  function createSignature(data) {

	var crypto = require('crypto');
	var keys = {
		priv: '-----BEGIN EC PRIVATE KEY-----\n' +
			'MHcCAQEEIE5dB89WjaMCHj3vtJuLxzFv66d5PxU5dLhUytLrqMGPoAoGCCqGSM49\n' +
			'AwEHoUQDQgAEuzel9jr4MlgciSnsP5FpI5sy4zXbVPzYjas2zWhxlVDdtILm+JTp\n' +
			'6zsBSp4Vcb5XEI2MHH85FAn5Y9GjgZk9Ig==\n' +
			'-----END EC PRIVATE KEY-----\n',
		pub: '-----BEGIN PUBLIC KEY-----\n' +
			'MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEuzel9jr4MlgciSnsP5FpI5sy4zXb\n' +
			'VPzYjas2zWhxlVDdtILm+JTp6zsBSp4Vcb5XEI2MHH85FAn5Y9GjgZk9Ig==\n' +
			'-----END PUBLIC KEY-----\n'
	};
	
	// Hash data
	var hash = crypto.createHash('sha256');
	hash.update(data);
	var hashedData = hash.digest('hex');
	console.log(hashedData );
	
	// sign data
	// signature is DER format started with 0x30
	var sign = crypto.createSign('sha256');
	sign.update(hashedData);
	var signature = sign.sign(keys.priv);
	console.log(signature);
	
	// verification
	var verify = crypto.createVerify('sha256');
	verify.update(hashedData);
	console.log(verify.verify(keys.pub, signature));
	return signature;
  }
```

### C# - using System.Security.Cryptography
For dot net,  Microsoft has System.Security.Cryptography lib used to cryptographic services, including secure encoding and decoding of data, as well as many other operations, such as hashing, random number generation, and message authentication. It is similar as node.js crypto lib. However, it defines  own key structure - CngKey, which defines the core functionality for keys that are used with Cryptography Next Generation (CNG) objects.   

However, it uses its own format for storing keys:

```c#
typedef struct _BCRYPT_ECCKEY_BLOB {
  ULONG dwMagic;
  ULONG cbKey;
} BCRYPT_ECCKEY_BLOB, *PBCRYPT_ECCKEY_BLOB;
```

This structure is used as a header for an elliptic curve public key or private key BLOB in memory.
```c#
BCRYPT_ECCKEY_BLOB
BYTE X[cbKey] // Big-endian.
BYTE Y[cbKey] // Big-endian.
```
The X and Y coordinates are unsigned integers encoded in big-endian format.

```c#
BCRYPT_ECCKEY_BLOB
BYTE X[cbKey] // Big-endian.
BYTE Y[cbKey] // Big-endian.
BYTE d[cbKey] // Big-endian.
```
The X and Y coordinates and d value are unsigned integers encoded in big-endian format.

It didn't design to take an external private key, but you can export a public key that can be used in other side to verify the signature. 

```c#
private byte[] CreateSignature(string data)
{
	CngKeyCreationParameters keyCreationParameters = new CngKeyCreationParameters();
	keyCreationParameters.ExportPolicy = CngExportPolicies.AllowExport;
	keyCreationParameters.KeyUsage = CngKeyUsages.Signing;
	
	CngKey key = CngKey.Create(CngAlgorithm.ECDsaP256, null, keyCreationParameters);
	byte[] publicKey = key.Export(CngKeyBlobFormat.EccPublicBlob);
	
	// Digital Signature Algorithm
	// Unfortunately signature is not DER format,
	// CND exports a concatenation of the two values and r is the first half of the array and 
	// s is the second half.
	ECDsaCng dsa = new ECDsaCng(key);
	byte[] dataBytes = Encoding.UTF8.GetBytes(data);
	var signature = dsa.SignData(dataBytes);   
	
	// now verify data has not been changed
	ECDsaCng ecsdKey = new ECDsaCng(key);
	var valid = ecsdKey.VerifyData(dataBytes, signature);
	return signature;
}
```

If verify the signature with OpenSSL, need convert the format to DER format. [Open source](https://github.com/dotnet/corefx/issues/21833) has implemented it.

### C# - using Bouncy Castle library
It is open source under MIT license, that can be used in Java and C# and compatible with OpenSSL. And signature is DER format.

```c#
private string CreateSignature(byte[] message)
{
	var key = FromHexString(PrivKey);
	var keyInt = new Org.BouncyCastle.Math.BigInteger(key);
	
	// secp256r1 == prime256v1
	var curve = SecNamedCurves.GetByName("secp256r1");
	var domain = new ECDomainParameters(curve.Curve, curve.G, curve.N, curve.H);
	var keyParameters = new ECPrivateKeyParameters(keyInt, domain);
	
	ISigner signer = SignerUtilities.GetSigner("SHA-256withECDSA");
	signer.Init(true /* forSigning */, keyParameters);
	signer.BlockUpdate(message, 0, message.Length);
	var signature = signer.GenerateSignature();
	
	// For test - verify signature
	var isValid = this.VerifySignature(message, signature);
	return BitConverter.ToString(signature).Replace("-", string.Empty);
}

private bool VerifySignature(byte[] message, byte[] signature)
{
	var key = FromHexString(PrivKey);
	var keyInt = new Org.BouncyCastle.Math.BigInteger(key);
	
	var curve = SecNamedCurves.GetByName("secp256r1");
	var domain = new ECDomainParameters(curve.Curve, curve.G, curve.N, curve.H);
	ECPoint q = domain.G.Multiply(keyInt);
	var publicParams = new ECPublicKeyParameters(q, domain);
	
	// check if public is the same as what generated from openSSL
	var pubKeyBytes = publicParams.Q.GetEncoded();
	
	var signer = SignerUtilities.GetSigner("SHA-256withECDSA");
	signer.Init(false /* forSigning */, publicParams);
	signer.BlockUpdate(message, 0, message.Length);
	
	return signer.VerifySignature(signature);
}
```

## DER signature format
We use DER signature format in our project. I found [the good article](https://superuser.com/questions/1023167/can-i-extract-r-and-s-from-an-ecdsa-signature-in-bit-form-and-vica-versa) explained this format:

```python
PS E:\certificate> xxd -i < sign.bin
0x30, 0x2d, 0x02, 0x14, 0x22, 0xd0, 0x8b, 0xc1, 0x0d, 0x0b, 0x7b, 0xff,
0xe6, 0xc1, 0x77, 0xc1, 0xdc, 0xc4, 0x2f, 0x64, 0x05, 0x17, 0x71, 0xc8,
0x02, 0x15, 0x00, 0xdd, 0xf4, 0x67, 0x10, 0x39, 0x92, 0x1b, 0x13, 0xf2,
0x40, 0x20, 0xcd, 0x15, 0xe7, 0x6a, 0x63, 0x0b, 0x86, 0x07, 0xb6
```
In this particular DER encoded signature, there's 47 bytes. A DER or PEM encoded signature can be inspected with OpenSSL's asn1parse command to find out what the bytes are:

```python
PS E:\certificate> openssl asn1parse -inform DER -in sign.bin
 0:d=0  hl=2 l=  45 cons: SEQUENCE          
 2:d=1  hl=2 l=  20 prim: INTEGER  :22D08BC10D0B7BFFE6C177C1DCC42F64051771C8
24:d=1  hl=2 l=  21 prim: INTEGER  :DDF4671039921B13F24020CD15E76A630B8607B6
```
In short, it says:
* At byte 0, there is a DER header of length 2, indicating a there's a SEQUENCE of elements to follow, with a total combined length of 45 bytes.
* At byte 2, there is a DER header of length 2, indicating an INTEGER element of length 20, the first element of the SEQUENCE (20 bytes are then printed in hex)
* At byte 24, there is a DER header of length 2, indicating an INTEGER element of length 21, the second element of the SEQUENCE (20 bytes are then printed in hex)

```python
PS E:\certificate> xxd -i < sign.bin
0x30, 0x2d, # Sequence of length 45 to follow (45 == 0x2d)
0x02, 0x14, # Integer of length 20 to follow (20 == 0x14)
```

* Here come the 20 bytes:
```python
0x22, 0xd0, 0x8b, 0xc1, 0x0d, 0x0b, 0x7b, 0xff, 0xe6, 0xc1, 
0x77, 0xc1, 0xdc, 0xc4, 0x2f, 0x64, 0x05, 0x17, 0x71, 0xc8,
0x02, 0x15, # Integer of length 21 to follow (21 == 0x15)
0x00,       # The first of the 21 integer bytes (see explanation below!)
```
* Here come the remaining 20 bytes
```python
0xdd, 0xf4, 0x67, 0x10, 0x39, 0x92, 0x1b, 0x13, 0xf2, 0x40, 
0x20, 0xcd, 0x15, 0xe7, 0x6a, 0x63, 0x0b, 0x86, 0x07, 0xb6
```