# Pulsar End To End Encryption
 * **Status**: Initial Draft
 * **Issue**: [633](https://github.com/apache/incubator-pulsar/issues/633)

## Introduction
Many systems use Pulsar to move messages between system components. Many of these systems publish, or would like to publish, messages that contain sensitive information. Across Pulsar clusters, network capture tools can be used to read these messages; in addition anyone with access to the persistent storage layer, or administrative access to Pulsar, can read all messages. Currently clients that wants to encrypt published messages have to implement their own solution outside of Pulsar client. It would be a better solution if Pulsar itself supports End To End Encryption as a feature. Once implemented, Pulsar clients have the option to protect their data by encrypting at the producer and decrypting at the consumer, which will prevent anyone other than the consumer with the appropriate key from decrypting the data. 

## Requirements
 * Support optional end-to-end encryption of messages published to Pulsar.
 * Message metadata is not encrypted by this mechanism.
 * Making encryption mandatory would break existing clients.
 * Avoid requiring shared secrets between message producers and consumers.
 * Support delivering a message to more than one consumers without requiring them to share any secrets.
 * Require no changes to the serving components of Pulsar.
 * Avoid adding key management infrastructure to the Pulsar client libraries.
 * The Pulsar client interface should allow alternate implementations to what is described in this document.

## Design
### Producer
The Pulsar client is modified so that a producer object may have one or more keys bound to it. At a minimum, the API will require a key name, the key content, and the key type (for future extensibility).
 * The user should be able to dynamically alter the set of bound keys.
 * If a producer has keys associated with it, before sending a message it will,
    * Generate or use a session with for the configured symmetric encryption function.
    * Encrypt the message body with the session key.
    * Encrypt the session key with each bound public key and optionally generate a message digest.
 * Producer should refresh the session key after N messages or every 4 hours(TBD: Make it configurable)
### Consumer
The Pulsar client is modified so that a consumer object may have one or more keys bound to it for decryption. These would be the private keys corresponding to the public ones given to the producer.  The user should be able to dynamically alter the set of bound keys.
 * The Pulsar consumer will look for a encryption_algo metadata in each message received.
 * Search the message metadata encryption_keys and look for one of the bound keys.
 * Decrypt the session key.
 * Decrypt the message using the session key if supported by the chosen encryption algorithm

## Client Usage
### Encrypting messages produced by Pulsar client:

1. Generating keys
    1. Generate rsa key pair using openssl.<br>`E.g: openssl genrsa -out testkey.pem 2048`
    1. Write private key in pkcs8 format.<br>`E.g: openssl pkcs8 -topk8 -inform PEM -outform PEM -in testkey.pem -out test_privkey.pem -nocrypt`
    1. Write public key in pkcs8 format.<br>`E.g: openssl rsa -in test_privkey.pem -pubout -outform pkcs8 -out test_pubkey.pem`
    1. Store the public and private key into a file or keystore. The key management and distribution is outside the scope of Pulsar.
1. Distribute the public keys to producer hosts and private keys to consumer hosts. Make sure the process has access to retrieve the key from the keystore.
1. Add keys to the ProducerConfiguration:
    1. Create ProducerConfiguration:<br>`ProducerConfiguration conf = new ProducerConfiguration()`
    1. Add key to the config:<br>`conf.addEncryptionKey(“pub-key-name”)`
<br>In some cases, the producer may want to encrypt the session key using multiple key individually and have them published with the corresponding key name in the message. Call `conf.addEncryptionKey(“pub-key-name”)` with the keyname to add them to the producer config. Consumer will be able to decrypt the message, as long as it has access to at least one of the keys.
1. Implement the callback which will be invoked by Pulsar client to load the key. Make sure not to perform any blocking operation within the callback, as it will block producer creation. The reason to get the key value using callback is to allow the producer to dynamically refresh the key when it expires. 
<br>`byte[] getEncryptionKey(String keyName)`
1. Create producer with the producer config. During the creation, the client will invoke the callback method for each key added to the producer config. Failing to retrieve a key will result in InvalidCryptoKeyException
<br>`PulsarClient client = PulsarClient.create("pulsar://localhost:6650");`
<br>`Producer producer = client.createProducer("persistent://property/cluster/ns/topic", conf);`
<br>`producer.send(msg);`
1. Handling publish failures when key is not present/ expired
    1. When the key is missing, send() will return InvalidCryptoKeyException
    1. If key is valid, but encryption failed due to Crypto library, send() will return CryptoException

### Decrypting encrypted messages received by consumer:

1. Make sure the consumer host has access to retrieve the key from keystore.
1. Create consumer and implement callback to retrieve key:
    1. Create ConsumerConfiguration:
<br>`ConsumerConfiguration conf = new ConsumerConfiguration()`
1. Implement the callback which will be invoked by Pulsar client to load the key when a key appears in a message. When one or more key appears in the message, Pulsar client assumes that the message is encrypted. Make sure not to perform any blocking operation within the callback, as it will block receive().
<br>`byte[] getEncryptionKey(String keyName)`
1. Create consumer with the consumer config
<br>`PulsarClient client = PulsarClient.create("pulsar://localhost:6650");`
<br>`Consumer consumer = client.subscribe("persistent://property/cluster/ns/topic", "subscription-name", conf);`
<br>`Message msg = consumer.receive();`
1. Handling failures when failed to decrypt message or when key is not present/ expired
    1. When the key is missing, receive() will return InvalidCryptoKeyException
    1. If key is valid, but decryption failed due to Crypto library, receive() will return CryptoException

### Handling symmetric and asymmetric keys
 * **Asymmetric encryption:**
By default Pulsar supports asymmetric key encryption using RSA keypair, as a result there is no need to share the secret with everyone. The public key is used by the producers to encrypt and produce the message. Only the person with the private key(in this case the consumer) will be able to decrypt the message.
 * **Symmetric encryption:**
Pulsar does not support symmetric key encryption at the moment.

### Revoking/ Invalidating key
1. If a key is revoked/ invalidated
    1. If key is revoked/invalidated, application should call conf.removeEncryptionKey(“key-name”) to prevent any producer from using it and recreate the producer.
    1. If consumer does not find the key corresponding to the one mentioned in the message, it would invoke the callback to refresh it. If it fails to get a valid key, decryption fails with exception. Consumer will attempt to do this for every message received.
1. If a key is refreshed
    1. Though it’s not recommended to modify an existing key, if a key needs to be refreshed, application should delete old producer object and create a new one. As long as the new value is returned when getEncryptionKey() is called, producer will use the refreshed key to encrypt the message.
    1. If consumer is already connected and processing messages, it does not refresh the key until it notices the session key change. Upon receiving updated session key,  consumer will call getEncryptionKey() to refresh the key. If decryption fails even with the refreshed key, receive() will fail with InvalidCryptoKeyException.
    1. If a new consumer is connected which only has access to the refreshed key, and incoming messages contains messages encrypted with old session as well as new session key, consumer won’t be able to decrypt older messages. Clients can set consumer configuration to control the behavior at this point.

## Supported Cipher suites:
 * **Key Strength recommendation:**
Cryptographic strength of RSA-2048 & ECDSA-
 * **Data encryption algorithm:**
AES-256-GCM

## Supported Crypto Libraries:
 * **Java: SunJCE**
 * **C++: OpenSSL**

## Failures and Exceptions:
### Producer:
1. InvalidCryptoKeyException - Encryption key not found
1. CryptoException - Encryption failed

### Consumer:
1. InvalidCryptoKeyException - Decryption key not found
1. CryptoException - Key is provided, but decryption failed
1. In some cases, consumer does not have access to the key at the time of receiving the message or client would like to receive the encrypted messages and store them for later processing. In such cases, getEncryptionKey() returns empty byte array, so consumer won’t be able to decrypt the message. Such messages are delivered as is to the application. It is the application’s responsibility to decrypt the message.
1. Received message is not encrypted.
By default receive() will fail with CryptoException. To ignore this failure and continue to receive messages, set <br>`conf.setAcceptNonEncryptedMessages(true)`
1. Encrypted messages received by a client which does not support encryption.
Earlier version of pulsar client does not even know that the message is encrypted, hence it would simply deliver the encrypted message to the consumer. So, it is upto the application to ensure that producer and consumer is configured properly with the appropriate keys and certified client version.

## MessageCrypto class to perform crypto actions
1. The key-name and key cipher are stored in an in-memory data structure. 
1. It does not persist keys. So, nothing is retained across restarts.
1. In memory data structure only keeps the encrypted key and not the actual key

## New Fields in MessageMetadata
1. encryption_algo
    * Describes the algorithm(s) to decrypt and verify the message. Ex: aes-256-gcm or aes-256-sha
1. encryption_keys
    * Key/Value pair of key name used to encrypt the session key and encrypted session key
1. encryption_param
    * Depends upon the value of `encryption_keys`
    * Additional inputs besides the key, required to decrypt and verify the message. For example, with aes-256-gcm, this property would contain the IV to decrypt the message and the authentication tag to verify it.