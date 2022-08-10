# Ruby Cryptography via OpenSSL

Anyone who’s worked with me before knows that I have a small security ‘thing’. I love cryptography, and I love to try to engineer the ‘unbreakable fortress’ of a system. Now we all know such a thing is impossible, but we should all try to do better at securing a system.
Now, I’ve learned a lot about cryptography from my time at the NASDAQ, and I try to use it heavily wherever warranted. One of my biggest fears when moving from C# to Ruby was the apparent lack of good documentation on cryptography. Most rubyiests never seem to surpass the MD5 password hash part of the endeavor so here I’ll present a simple beginning to secure Ruby-on-Rails

## Symmetric Encryption
Symmetric encryption is called symmetric because the process of encryption and decryption are inversions of the same operation. Say for instance, I have a table of each letter in the alphabet and a corresponding table of random letters. School children use this method all the time. They use the table to translate each letter in the message to the corresponding random letter. Having shared the table before hand, the person receiving the message would simply look up the letters in the second column to find their original value. This is symmetric because the ‘key’ (the table of letters) is known by both parties and the decrypt operation is simply to inverse the encrypt operation.
In computer science we have many symmetric algorithms to choose from. DES, AES, TripleDES, Blowfish and many others are available for use. Some are better then others, and a comparison of the different algorithms is beyond the scope of this article, but its safe to say that the American Encryption Standard (AES), is sufficiently strong, and interoperable for most purposes. In addition to algorithm type, many algorithms also support different key lengths. Larger key lengths provide better security then shorter ones, and are usually measured in number of bits. This leads to names like AES-256 (AES, with a 265 bit key).
Once the algorithm and length are determined, you can then perform encryption and decryption operations. Some algorithms provide another value known as the IV or initialization vector. This value is a random block of information to ensure that even when encrypting the same message twice the cypher texts are not the same. The IV is often transmitted with the message, and contains no secure information.

```ruby
require 'openssl'
cipher = OpenSSL::Cipher.new('AES256')
# Set into 'encrypt mode'
cipher.encrypt
key = cipher.random_key
iv = cipher.random_iv
cipher.key = key
cipher.iv = iv
message = 'This is a secure message,
meet at the clock-tower at dawn.'
cyphertext = cipher.update message
cyphertext += cipher.final
# I would then transmit the cypher-text and IV
cipher.reset
cipher.decrypt
cipher.key = key
cipher.iv = iv
plaintext = cipher.update cyphertext
# The plaintext should now be the same
# as the original message
 puts plaintext
 ```

## Public Key Cryptography

We all know that we’re ‘secure’ when communicating over SSL, but why? Well, here’s a breakdown of the process from 1000ft.
1. We connect to the server on port 443 and request a secure connection
2. The server presents it’s “certificate” saying who it is and who says that it’s telling the truth
3. We verify that the certificate was indeed ‘signed’ by a party we trust and that the name matches the URL we’re accessing
4. We encrypt some information using a “public key” listed on the certificate, since we verified the certificate, we’re sure that this value hasn’t been tampered with
5. The server uses its private key to read our encrypted data, and a two way session is established
That’s right, we were able to securely communicate with the server, and all that was required was that we have a 3rd party who is trusted to facilitate the process. The part of the process I want to focus on is two fold:
• Our verification of the certificate
• Transmission of data securely to the server

## Digital signatures

Digital signatures are interesting mathematical problems. They operate in a similar manner to a real signature, in that only one person can create one, but anyone can ensure that the signature is authentic. So what’s that have to do with math? Well there are a few mathematical problems that are easy to do, but hard, or impossible to undo. Modulus is an example of an undoable operation, take for instance that I calculate 12 MOD 5 = 2. This was simple to perform, but it’s impossible to figure out what the input was given x MOD 5 = 2, primarily because there are multiple solutions. I could have simply calculated 2 MOD 5 = 2. Public key cryptography works somewhat in the same way. The most common used cypher on SSL is called RSA after the creators. It’s interesting mathematical property is that if you find two extremely large prime numbers (a few hundred digits in length) and multiply them together, it is difficult to figure out what the prime factors of the resulting number were.
1.
2.
3.
4. 5.
I create a private key of two large primes, called p and q, I save these values securely and tell no one what they are
I multiply the primes together to create a huge value N, this is my public key. I publish this and tell as many people as possible.
Once these two values are created, I am able to take a value M and perform the signing operation with p and q, this produces a signature value S.
I publish M and S on the internet
Someone who wishes to verify that it was indeed me who published M and can be sure they have the correct value for N can perform a verify operation given M, S, and N to tell them whether the values are authentic.
Without detailing the exact nature of the RSA algorithm it comes down to something a bit like this (I am omitting some detail about cofactors etc. for simplicity, there are better write- ups of the full algorithm anyway):
Now, how did this work with SSL? Well, the certificate is a few things, namely the domain name of the site, a public key for encryption, a declaration of encryption cyphers used etc. All of that information is then signed by a trusted third party, who’s value is included in the web browser or operating system. These values are from companies such as Verisign and GoDaddy, provide the trusted third party in the mix.
So now I hope you have a basic idea of the concept... here’s what it looks like in code.

```ruby
 require 'openssl'
require 'base64'
# 4096 is a somewhat arbitrary value defining
# the size of the key. Larger values are
# more secure, but take more time to use
private_key = OpenSSL::PKey::RSA.new(4096)
public_key = private_key.public_key
# Here we print some values for reference
puts "Private Key: " +
      Base64.encode64(private_key.to_der)
puts "Public Key: " +
      Base64.encode64(public_key.to_der)
my_stuff = 'This is Richard Penwell speaking'
signature = private_key.private_encrypt my_stuff
if my_stuff ==
    public_key.public_decrypt(signature)
  puts "Signature verified"
else
  puts "Signature failes"
end
```

```ruby
require 'openssl'
require 'base64'
# 4096 is a somewhat arbitrary value defining
# the size of the key. Larger values are
# more secure, but take more time to use
# This generates Bob's key
private_key = OpenSSL::PKey::RSA.new(4096)
public_key = private_key.public_key
# Here we print some values for reference
puts "Private Key: " +
    Base64.encode64(private_key.to_der)
puts "Public Key: " +
    Base64.encode64(public_key.to_der)
# These lines are executed
# in the context of Alice
my_stuff = 'This is top secret'
cyphertext = public_key.public_encrypt my_stuff
# The below is executed in the context of Bob
plaintext =
    private_key.private_decrypt cyphertext
assert my_stuff == plaintext
```

Asymmetric encryption uses similar mathematics to digital signatures, but for a different purpose. In public key cryptography, we are able to publish a public key, for which only the private key can decrypt data. Using the archetypal Alice, Bob, and Eve, the story proceeds as follows:
1. Alice and Bob meet for coffee at the local Starbucks. They have a wonderful conversation and decide that should they want to discuss anything further they should get in touch. Both of them have calculated a private and a public key. They write down their public key and give it to the other and bid each other farewell.
2. Alice gets home and realizes she never gave Bob that last critical piece of information, so she pulls Bob’s key out of her pocket, and encrypts a message to that public key
3. What’s interesting at this point is that not even Alice, with knowledge of the public key, cyphertext, and plaintext, can she derive the original plaintext, nor Bob’s private key.
4. For good measure, she uses her private key to “sign” the encrypted message as well.
5. She takes the encrypted message, and signature and sends them to Bob.
6. Bob receives the message, and is able to use his private key to decrypt the message. In addition, he uses Alice’s public key to verify that it was indeed from Alice.
7. Eve, who is assumed to be able to obtain and modify the message en route would be limited to only knowing the source, destination, possibly the size, and time the message was sent. She may be able to stop the message from going through, but was prevented from reading, or modifying the message.
Here’s a modified version of the above to accomplish this:

## Cryptographic Performance
Public key cryptography is typically an expensive operation. Generating a new RSA key with a length of 4096 takes 11 seconds on my one year old MacBook pro. In order to improve performance of public key encryption over large messages, we typically combine the symmetric and asymmetric methods. This is done by creating a symmetric cypher with a random key, encrypting the data, and finally encrypting the random key using asymmetric methods.
This allows us to benefit from PKI while keeping stronger performance, especially over large data sets such as Video files and the like.