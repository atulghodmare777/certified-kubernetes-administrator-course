# TLS Basics
  - Take me to [Video Tutorial](https://kodekloud.com/topic/tls-basics/)
  
In this section, we will take a look at TLS Basics

## Certificate
- A certificate is used to guarantee trust between 2 parties during a transaction.
- Example: when a user tries to access web server, tls certificates ensure that the communication between them is encrypted.

  ![cert1](../../images/cert1.PNG)
  
  
## Symmetric Encryption
- It is a secure way of encryption, but it uses the same key to encrypt and decrypt the data and the key has to be exchanged between the sender and the receiver, there is a risk of a hacker gaining access to the key and decrypting the data.

  ![cert2](../../images/cert2.PNG)
  
## Asymmetric Encryption
- Instead of using single key to encrypt and decrypt data, asymmetric encryption uses a pair of keys, a private key and a public key.

  ![cert3](../../images/cert3.PNG)
  
  ![cert4](../../images/cert4.PNG)
  
  ![cert5](../../images/cert5.PNG)
  
  ![cert6](../../images/cert6.PNG)

- In this we create a ssh key at the client side using following command and it will create private and pub key
  ```
  ssh-keygen
  output: id_rsa id_rsa.pub
  ```
- Then paste the pub key in server /.ssh/authorized_key
- Then we can login in the server using command: ssh -i id_rsa user1@server1
- If want to access the other servers then we can copy same pub key and paste in the server location ( /.ssh/authorized_key) then we can access the server
  with same private key
- If other users want to access the server then they can generate their own ssh key and as a admin we can paste in server for access through ssh.
  So now they can access the server using their private key
- The problem with symmetric encryption is that we have to send the key to server with encrypted data which can be sniffed by hacker and we will be compromised,
  as he can decrypt using the same key.
- What if we could trasfer key to the server safely, by that way later client and server can communicate safely using symmetric encryption.
- To securely trasfer the key from client to server we use assymmetric encryption. We generate a pub and prv key on server. We use following command on server
  ```
  openssl genrsa -out my-bank.key 1024
  openssl rsa -in my-bank.key -pubout > mybank.pem
  ```
- When the user access the server first time he gets public key from the server( hacker also can get pub key), then users browser encrypt the symmetric key using
  pub key provided by the server. The symmentric key is now secure .Now the user sends this to the server( hacker also gets this key), Now the server use the        privat key and decrypt the symmentric key( However the hacker does not have a private key to retrieve the symmentric key, he only have pub key with which he can
  only lock or encrypt the message) . Now the symmentric key is safely available with user and server. Now user can use symmentric key to encrypt the data safely
  and server can use the same symmentric key to decrypt the data. With assymetric encryption we have successfully trasffered the symmetric key from user to server.
- Hacker can try to create the website exact replica where you will be typing the credentials, when hacker server sends a key which will be in certificate format,
  there we can find all the details correct with respect to genuine website exa:domain and everything, but it will be self signed not signed by certificate          authority(CA) by that way we can know its hacker, by default browsers come up with certification validation mechanism, if found fake cerificate then it will       warn us.
- There are well known CA which will sign and validate certicate. Exa: symantec, digicert, globalsign etc.
- How to get our certificate signed: We need to generate a certificate signing request(CSR) with our key generated earlier and the domain name of website,
  we can do this using following command:
  ```
  openssl req -new -key my-bank.key -out my-bank.csr -subj "/C=US/ST=CA/O=MyOrg, Inc./CN=my-bank.com"
  output: my-bank.key my-bank.csr
  ```
- The CA will verify the details and sign and send it back to us.
- But how browsers know the CA is ligitimate? Exa: How browser know that symentec is valid CA ?
  CA's themselves will have their own pub and private key pairs, CA's use their private keys to sign the certificates, pub keys of CA's will be built in with the    browsers, browser uses the pub key of the CA's to validate the certificate actually signed by the CA themselves. We can see the CA in the browser settings in      the certificates section under trusted CA's tab.
- Public CA's wont help websites which are hosted privately within the organisation. Exa: Accessing payrol or internal mail application. For this we can host        private CA's. These CA's also provide private offerings as well. The CA server we can deploy in a server internally, then we install pub key of our CA in all
  the browsers of employees, and we can establish secure connection within the organisation.
-  
  

#### How do you look at a certificate and verify if it is legit?
- who signed and issued the certificate.
- If you generate the certificate then you will have it sign it by yourself; that is known as self-signed certificate.

  ![cert7](../../images/cert7.PNG)
  
#### How do you generate legitimate certificate? How do you get your certificates singed by someone with authority?
- That's where **`Certificate Authority (CA)`** comes in for you. Some of the popular ones are Symantec, DigiCert, Comodo, GlobalSign etc.

  ![cert8](../../images/cert8.PNG)
  
  ![cert9](../../images/cert9.PNG)
  
  ![cert10](../../images/cert10.PNG)
  
## Public Key Infrastructure
   
   ![pki](../../images/pki.PNG)
   
## Certificates naming convention

  ![cert11](../../images/cert11.PNG)
  
  

  
   

  
  
  

  
  
  
  
  
  

  
  
