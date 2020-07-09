# Padding Oracle Attack Explained
Padding Oracle attack fully explained and coded from scratch in Python3.

------ Page Under Construction -------


### Summary  

1- Overview   
2- Script Usage    
3- AES-CBC Ciphering    
4- Exploiting CBC mode    
5- Padding Oracle Attack    

* * *
## 1- Overview

The padding oracle attack is a spectacular attack because it allows to decrypt a message that has been intercepted if the message was encrypted using CBC mode. 
It will only require ensuring that we are able to obtain a response from the server that will serve as an Oracle (we'll come back to these in more detail later in this report). We will then be able to decrypt the entire message except the first block, un less you know the initialization vector.   

In this article, we will focus on how to use this vulnerability and propose a python script that exploits CBC mode to decrypt a message encrypted in AES-CBC.

* * *
## 2- Script Usage

If you're only insterested in using the code, the chapter 2 is all you need. However, please note that this code consider that you know the initialization vector, which is usually wrong in real life

Get the program by downloading this repository or:
~~~
$ git clone https://github.com/flast101/padding-oracle-attack-explained.git
~~~

Cryptographic parameters can be changed in `settings.py`

Encyption and decryption using AES-CBC alogorithm:
~~~
$ python3 aescbc.py <message>         encrypts and displays the message (output in hex format)
$ python3 aescbc.py -d <hex code>      decrypts and displays the message
~~~

Decrypting an message using the padding oracle attack:
~~~
$ python3 poracle_exploit.py <message>         decrypts and displays the message
~~~

`oracle.py` is our oracle: a boolean function determining if the message is encrypted with valid PKCS7 padding .


## Example

~~~
root@kali:~# python3 poracle_exploit.py dfd117358343ca9b36e58abec333349d753937af1781b532404c8b29b25d4de24661995fb5dcb06528a15b4eed172d7410c28b5f38cd0af834afdbe5b9ff36a1c516c8a1cb7ad4e32a122ea918aeca60
Decrypted message:  Try harder ! The quieter you become the more you are able to hear.
~~~



![example.png](images/example.png "example.png")




* * * 
## 3- AES-CBC Ciphering
### 3.1- Advanced Encryption Standard (AES)
Safeguarding information has become an indispensable measure in today’s cybersecurity world. Encryption is one such method to protect discreet information being transferred online.

The Advanced Encryption Standard (AES), also known by its original name Rijndael (Dutch pronunciation: [ˈrɛindaːl]),[3] is a specification for the encryption of electronic data established by the U.S. National Institute of Standards and Technology (NIST) in 2001.

The Encryption technique is employed in two ways, namely symmetric encryption and asymmetric encryption. For symmetric encryption, the same secret key is used for both encryption and decryption, while asymmetric encryption has one key for encryption and another one for decryption.

With regard to symmetric encryption, data can be encrypted in two ways. There are stream ciphers: any length of data can be encrypted, and the data does not need to be cut. The other way is block encryption. In this case, the data is cut into fixed size blocks before encryption.

There are several operating modes for block encryption, such as Cipher Block Chaining (CBC), as well as CFB, ECB... etc.



### 3.2- Cipher Block Chaining (CBC)
In CBC mode, each block of plaintext is XORed with the previous ciphertext block before being encrypted. This way, each ciphertext block depends on all plaintext blocks processed up to that point. To make each message unique, an initialization vector must be used in the first block. 

CBC has been the most commonly used mode of operation, in applications such as VPN with OpenVPN or IPsec. Its main drawbacks are that encryption is sequential (i.e., it cannot be parallelized), and that the message must be **padded** to a multiple of the cipher block size.

![cbc_mode.png](images/cbc_mode.png "cbc_mode.png")

If the first block has the index 0, the mathematical formula for CBC encryption is:

**C<sub>i</sub> = E<sub>K</sub> (P<sub>i</sub> ⊕ C<sub>i-1</sub>) for i ≥ 1,     
C<sub>0</sub> = E<sub>K</sub> (P<sub>0</sub> ⊕ IV)**

Where E<sub>K</sub> is the function of encryption with the key K and C<sub>0</sub> is equal to the initialization vector.


Decrypting with the incorrect IV causes the first block of plaintext to be corrupt but subsequent plaintext blocks will be correct. This is because each block is XORed with the ciphertext of the previous block, not the plaintext, so one does not need to decrypt the previous block before using it as the IV for the decryption of the current one. This means that a plaintext block can be recovered from two adjacent blocks of ciphertext. 






* * *
## 4.- Exploiting CBC mode
### 4.1- PKCS7 padding validation function

The padding mainly used in block ciphers is defined by PKCS7 (Public-Key Cryptography Standards) whose operation is described in RFC 5652.   
Let N bytes be the size of a block. If M bytes are missing in the last block, then we will add the character ‘0xM’ M times at the end of the block.

Here, we want to write a function which takes as input clear text in binary and which returns a boolean validating or invalidating the fact that the fact that this text is indeed a text with a padding in accordance with PKCS7.   
The function is exposed in the code which follows under the name **_pkcs7_padding_**. It determines whether the input data may or may not meet PKCS7 requirements.

```python
def pkcs7_padding(data):
    pkcs7 = True
    last_byte_padding = data[-1]
    if(last_byte_padding < 1 or last_byte_padding > 16):
        pkcs7 = False
    else:
        for i in range(0,last_byte_padding):
            if(last_byte_padding != data[-1-i]):
                pkcs7 = False
    return pkcs7
```

### 4.2- Ask the Oracle

Here, we want to perform a function that determines whether an encrypted text corresponds to PKCS7 padding valid encrypted data. This Oracle function will abundantly serve us in the manipulation allowing to exploit the fault due to padding.

```python
def oracle(encrypted):
    return pkcs7_padding(decryption(encrypted))
```

### 4.3- CBC mode vulnerability

Let's take a theoretical example, a character string which, when padded, is made of 4 blocks of 16 bytes each. The 4 plaintext blocks are P<sub>0</sub> to P<sub>3</sub> and the 4 encrypted blocks are C<sub>1</sub> to C<sub>3</sub>.

We can illustrate it with the following diagram:

![four_blocks.png](images/four_blocks.png "four_blocks.png")

We wrote this formula in th eprvious chapter:   
**C<sub>i</sub> = E<sub>K</sub> (P<sub>i</sub> ⊕ C<sub>i-1</sub>)**

If we pply decryption on both side of the formula, it gives    
**D<sub>K</sub> ( C<sub>i</sub> ) = P<sub>i</sub> ⊕ C<sub>i-1</sub>**    

And thanks to XOR properties:
**P<sub>i</sub> = D<sub>K</sub> ( C<sub>i</sub> ) ⊕ C<sub>i-1</sub>** 
 


Now let's take a totally random new X block. It's a block that we create and that we that we can change. Let's take with it the last encrypted block from our example, C<sub>3</sub>, and concatenate them.

It gives the following Diagram:

![two_blocks.png](images/two_blocks.png "two_blocks.png")


Applying our maths to this diagram, we can write the 2 following formulas:

- C<sub>3</sub> = E<sub>K</sub> ( P<sub>3</sub> ⊕ C<sub>2</sub> )
- P'<sub>1</sub> = D<sub>K</sub> ( C<sub>3</sub> ) ⊕ X

Now, we can replace "C<sub>3</sub>" by "E<sub>K</sub> ( P<sub>3</sub> ⊕ C<sub>2</sub> )" in the second formula:   
**P'<sub>1</sub> = P<sub>3</sub> ⊕ C<sub>2</sub> ⊕ X**

We have something really interesting here because this fromula is the link between 2 known elements and 2 unknown elements.

**Known elements:**
- X: this is the element that we control, we can choose it.
- C<sub>2</sub>: this is the penultimate encrypted block.

**Unknown elements:**
- P<sub>3</sub>: the last plaintext block, which we are trying to find.
- P'<sub>1</sub>: the plaintext block coming from the concatenation of X and C<sub>3</sub>, and which depends on padding mechanism. We don't know it, but we will discover it thanks to the padding in the next xchapter.

**More importantly, this equation has no cryptography anymore, only XOR. We could skip the cryptographic aspect only with math.**

This is exactely where resides the vulnerability of CBC mode... and the beauty of this attack. Using math, we have just demonstrated that we can get rid of cryptography  if we know how PKCS7 padding works.

* * *
## 5- Padding Oracle Attack
### 5.1- Last byte

We just saw that    
P'<sub>1</sub> = P<sub>3</sub> ⊕ C<sub>2</sub> ⊕ X

As XOR operation is commutative, the following formula is also true:
**P<sub>3</sub> = P'<sub>1</sub> ⊕ C<sub>2</sub> ⊕ X**

This equality only contains the XOR operation. As you know, the XOR is a bit by bit operation, so we can split this equality by calculating it byte by byte.
Our blocks size are 16 bytes, we have the following equations:   
P<sub>3</sub>[0] = P'<sub>1</sub>[0] ⊕ C<sub>2</sub>[0] ⊕ X[0]   
P<sub>3</sub>[1] = P'<sub>1</sub>[1] ⊕ C<sub>2</sub>[1] ⊕ X[1]    
P<sub>3</sub>[2] = P'<sub>1</sub>[2] ⊕ C<sub>2</sub>[2] ⊕ X[2]  
(...)   
P<sub>3</sub>[14] = P'<sub>1</sub>[14] ⊕ C<sub>2</sub>[14] ⊕ X[14]   
P<sub>3</sub>[15] = P'<sub>1</sub>[15] ⊕ C<sub>2</sub>[15] ⊕ X[15]

We also know that the decryption of an encrypted text must be a plaintext with a valid padding, therefore ending with 0x01 or 0x02 0x02 etc.   
As we control all bytes of X, we can bruteforce the last byte (256 values between 0 and 255) until we obtain a valid padding, i.e. until the oracle function returns _"True"_ when its input is X + C<sub>3</sub> (concatenation of X and C<sub>3</sub>).     
**In this case, it will mean that the clear text padding of P’<sub>1</sub> ends with 0x01.**


Once we find the last byte of X which gives the valid padding, we know that the padding value P’_2 [15] = 0x01, which means:   
**P<sub>3</sub>[15] = P'<sub>1</sub>[15] ⊕ C<sub>2</sub>[15] ⊕ X[15] = 0x01 ⊕ C<sub>2</sub>[15] ⊕ X[15]**

With this information, we find the last byte of the last block of text plaintext (which is padding, but it's a good start)!

### 5.2- What else ?

Now, we will look for the value of the previous byte of P<sub>3</sub>, ie. P<sub>3</sub>[14] in our case.

We are going to assume here that we have a padding of 0x02 on P’<sub>1</sub>, which results in P’<sub>1</sub>[15] = P’<sub>1</sub>[14] = 0x02. And by the way, our P<sub>3</sub>[15] is now known since we just found it.   

So this time we have the following:

- X[15] = P'<sub>1</sub>[15] ⊕ C<sub>2</sub>[15] ⊕ P<sub>3</sub>[15] = 0x02 ⊕ C<sub>2</sub>[​ 15​ ] ⊕ P<sub>3</sub>[15] where C<sub>2</sub>[15] et P<sub>3</sub>[15] are known
- P<sub>3</sub>[14] = P'<sub>1</sub>[14] ⊕ C<sub>2</sub>[14] ⊕ X[14] = ​ 0x02​ ⊕ C<sub></sub>[14] ⊕ X[14] 
 
It is therefore X[14] that we brute force, that is to say that we vary between 0 and 256 in hexa, with P'<sub>1</sub>[14] whose value is 0x02, to find an X + C<sub>3</sub> whose padding is valid.

We have all the values in hand which allow us to find P<sub>3</sub>[14], and ffter this step we know the last 2 bytes of P<sub>3</sub>, that is to say the plain text that interests us.

### 5.3- Generalize it.

This reasoning is to be looped until you find all the values ​​of the plaintext of the block.

Once the block has been decrypted, just take the next blocks and apply exactly the same reasoning. We will then find the blocks P<sub>2</sub>, and P<sub>1</sub>.

However, a problem arises in finding the block P<sub>0</sub>. Indeed, for the previous cases, the decryption was based on the knowledge of the encrypted block which preceded the block being
decrypted. However, for the first block, you must know the IV used. In this case, no miracle:

- Either you know the IV (Initiation Vector or Initialization Vector), in which case it's the same reasoning,
- Or you try to guess it using usual combinations, such as a null IV, or a series of consecutive bytes and you may or may not decrypt the last block.    

If we cannot find it, then we will have to settle for the decryption of blocks 1 to N-1.

### 5.4- One formula to rule them all.

We can notice that we have everything we need to decrypt the text but let's recap.

We have encrypted text which we know is encrypted in N blocks of size B. From there, we can split the encrypted text into N blocks where N = (encrypted message size) / B.    
If the message has been encrypted correctly, N is necessarily an integer.

We split the text into N blocks of B bytes and we start with the last byte of the last block. At this stage :    

P<sub>N-1</sub>[B-1] = P'<sub>1</sub>[B-1] ⊕ C<sub>N-2</sub>[B-1] ⊕ X[B-1] = 0x01 ⊕ C<sub>N-2</sub>[B-1] ⊕ X[B-1] 
where X[B-1] is the byte that satisfied the requirements of PKCS7.

**Then, we iterate on i between 0 and B-1 and on j between 0 and N-1. At each turn, we have:**


**- X[i+1] = P'<sub>1</sub>[i+1] ⊕ C<sub>j-1</sub>[i+1] ⊕ P<sub>j</sub>[i+1] = 0x02 ⊕ C<sub>j-1</sub>[i+1] ⊕ P<sub>j</sub>[i+1]**     
where C<sub>j-1</sub>[i+1] and P<sub>j</sub>[i+1] are known.     
**- P<sub>j</sub>[i] = P'<sub>1</sub>[i] ⊕ C<sub>j-1</sub>[i] ⊕ X[i] = 0x02 ⊕ C<sub>j-1</sub>[i] ⊕ X[i]**    
where X[i] is the byte that satisfied the requirements of PKCS7.




Happy hacking !   :smiley:

