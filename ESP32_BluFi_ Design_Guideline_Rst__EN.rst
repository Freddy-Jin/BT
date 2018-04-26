BLUFI
*****

Overview
========
BluFi for ESP32 is a network-configuration funcion via Bluetooth channel.

It is built on the GATT protocol, which defines the procedure of ESP32 working as the GATT Server to connect the GATT Client (e.g. an SoftAP created by a mobile phone) through Wi-Fi or configuring for the SoftAP Profile. Sharding, data encryption, checksum verification in the BluFi layer are the key parts that need your attention.

The algorithms of symmetric encryption, asymmetric encryption and checksum support custmization on your demand in practical use. Here we use the DH algorithm for key negotiation, 128-AES algorithm for data encryption, and CRC16 algorithm for checksum verification.

The BluFi Flow:
---------------
The BluFi networking flow includes the configuration of the SoftAP and Station.

Take the configuration of the Station as an example to illustrate the core parts of the procedure, including broadcast, connection, service discovery, negotiation of the shared key, data transmission, connection status backhaul.

The procedure is as follows:
------------------------------

1. Set the ESP32 into GATT Server mode and then it will send broadcasts with specific *adv data*. You can customize this broadcast as needed, which is not a part of the BluFi Profile.

2. Use the APP installed on the mobile phone to search for this particular broadcast. The mobile phone will connect to ESP32 as the GATT Client once the broadcast is confirmed. The APP used during this part is up to you.

3. After the GATT connection is successfully established, the mobile phone will send a data frame for key negotiation to ESP32 (see the section of "The Formats of Data Frames" for details).

4. After ESP32 receives the data frame of key negotiation, it will parse the content according to the user-defined negotiation method.

5. The mobile phone works with ESP32 for key negotiation using the encryption algorithms such as DH, RSA or ECC.

6. After the negotiation process is completed, the mobile phone will send a control frame for security-mode setup to ESP32.

7. When receiving this control frame, ESP32 will be able to encrypt and decrypt the communication data using the shared key and the security configuration.

8. The mobile phone sends the data frame defined in the section of "The Formats of Data Frames"，with the Wi-Fi configuration information to ESP32, including SSID, password, etc.

9. The mobile phone sends a control frame of Wi-Fi connection request to ESP32. When receiving this control frame, ESP32 will regard the communication of essential information as done and get ready to connect to the Wi-Fi.

10. After connecting to the Wi-Fi, ESP32 will send a control frame of Wi-Fi connection status report to the mobile phone，to report the connection status. At this point the networking procedure is completed.

.. note::

    1. After ESP32 receives the control frame of security-mode configuration, it will execute the operations in accordance with the defined security mode.

    2. The data lengths before and after symmetric encryption/decryption must stay the same. It also supports in-place encryption and decryption.

The flow chat of BluFi:

.. figure:: https://github.com/Freddy-Jin/ESP32_BLUFI_-Design_Guidelines/blob/master/Docs/Figure1.png
    :align: center
    :figclass: align-center

The Frame Formats Defined in BluFi
***************************************

The frame formats for the communication between the mobile phone APP and ESP32 is defined as follows:

The frame format with no fragment (8 bit)：

+------------+---------------+-----------------+-------------+----------------+----------------+
| LSB - Type | Frame Control | Sequence Number | Data Length | Data           | MSB - CheckSum |
+------------+---------------+-----------------+-------------+----------------+----------------+
| 1          | 1             | 1               | 1           | ${Data Length} | 2              |
+------------+---------------+-----------------+-------------+----------------+----------------+

If the **Frame Ctrl** bit is enabled, the **Total length** bit indicates the length of data frame's rest part. It can tell the remote how much memory needs to be alloced.

The frame format with fragments（8 bit）：

+------------+--------------------+----------------+------------+-------------------------------------------+----------------+
| LSB - Type | FrameControl(Frag) | SequenceNumber | DataLength | Data                                      | MSB - CheckSum |
+            +                    +                +            +-------------------------------------------+                +
|            |                    |                |            | Total Content Length | Content            |                |
+------------+--------------------+----------------+------------+----------------------+--------------------+----------------+
| 1          | 1                  | 1              | 1          | 2                    | ${Data Length} - 2 | 2              |
+------------+--------------------+----------------+------------+----------------------+--------------------+----------------+

Normally, the control frame does not contain data bits, except for Ack frame.

The format of Ack Frame（8 bit）：

+------------------+----------------+------------------+--------------+-----------------------+----------------+
| LSB - Type (Ack) | Frame Control  | SequenceNumber   | Data Length  | Data                  | MSB - CheckSum |
+                  +                +                  +              +-----------------------+                +
|                  |                |                  |              | Acked Sequence Number |                |
+------------------+----------------+------------------+--------------+-----------------------+----------------+
| 1                | 1              | 1                | 1            | 1                     | 2              |
+------------------+----------------+------------------+--------------+-----------------------+----------------+

1. Type

   Type field, taking 1 Byte, is divided into Type and Subtype that Type uses the lower 2 bit and Subtype uses the upper 6 bit.

   * The control frame is not encrypted for the time being and supports to be verified;

   * The data frame supports to be encrypted and verified.

2. Frame Control

   Control field, takes 1 Byte and each bit has a different meaning.

3. Sequence Control

   Sequence control field. When a frame is sent,the value of sequence fied is automatically added by 1 regardless of the type of frame, which prevents Replay Attack. The sequence is cleared after each reconnection.

4. Length

   The length of the data field that does not include CheckSum.

5. Data

   The instruction of the data field is different according to various value s of Type or Subtype. Please refer to the table above.

6. CheckSum

   This field takes 2 bytes that is used to check "sequence + data length + clear text data".

The Security Implementation of ESP32
*************************************

1. To secure data

   To ensure that the transmission of the Wi-Fi SSID and password is secure, the message needs to be encrypted using symmetric encryption algorithms, such as AES, DES and so on. Before using symmetric encryption algorithms, the devices are required to negotiate (or generate) a shared key using an asymmetric encryption algorithm (DH, RSA, ECC, etc).

2. To ensure data integrity

   To ensure data integrity, you need to add a checksum algorithm, such as SHA1, MD5, CRC, etc.

3. Identity security (signature)

  Algorithm like RSA can used to secure identity. But for DH, it needs other algorithms as an companion for signature.

4. To prevent replay attack

   It is added to the Sequence field and used during the checksum verification.

   For the coding of ESP32, you can determine and develop the security processing, such as key negotiation. The mobile application sends the negotiation data to ESP32 and then the data will be sent to the application layer for processing. If the application layer does not process it, you can use the DH encryption algorithm provided by BluFi to negotiate the key. The application layer needs to register several security-related functions to BluFi:

.. highlight:: none

::

   typedef void (*esp_blufi_negotiate_data_handler_t)(uint8_t *data, int len, uint8_t **output_data, int *output_len, bool *need_free);

   This function is for ESP32 to receive normal data during negotiation, and after processing is completed, the data will be transmitted using Output_data and Output_len.

   BluFi will send output_data from Negotiate_data_handler after Negotiate_data_handler is called.

   Here are two "*", because the length of the data to be emitted is unknown that requires the function to allocate itself (malloc) or point to the global variable, and to infrom whether the memory needs to be freed by NEED_FREE.


.. highlight:: none

::

   typedef int (* esp_blufi_encrypt_func_t)(uint8_t iv8, uint8_t *crypt_data, int cyprt_len);
    
   The data to be encrypted and decrypted must use the same length. The IV8 is a 8 bit sequence value of frames, which can be used as a 8 bit of IV.

.. highlight:: none

::

   typedef int (* esp_blufi_decrypt_func_t)(uint8_t iv8, uint8_t *crypt_data, int crypt_len);

   The data to be encrypted and decrypted must use the same length. The IV8 is a 8 bit sequence value of frames, which can be used as a 8 bit of IV.

.. highlight:: none

::

   typedef uint16_t (*esp_blufi_checksum_func_t)(uint8_t iv8, uint8_t *data, int len);

   This function is used to compute CheckSum and return a value of CheckSum. Blufi uses the returned value to compare the CheckSum of the frame.

GATT Related Instructions
*************************

UUID:
==========

BluFi Service UUID： 0xFFFF，16 bit

BluFi（the mobile -> ESP32）: 0xFF01, writable

Blufi（ESP32 -> the mobile phone）: 0xFF02, readable and callable

.. note::

	1. The Ack mechanism is already defined in the profile, but there is no code implementation.

	2. Other parts have been implemented.
