# PQRSTUV - Post Quantum Resource Station Tucked Under Vault

## DRAFT v1.

## DISCLAIMER: NOT FOR COMMERCIAL USE

This cryptographic standard is provided for educational, research, and experimental purposes only. It has not undergone the rigorous security audits and testing required for production environments.
DO NOT USE THIS SOFTWARE/STANDARD IN COMMERCIAL APPLICATIONS OR PRODUCTION SYSTEMS.

Using this software in commercial products or to protect sensitive data in production environments may expose you to significant security risks. Commercial-grade cryptographic implementations require:

- Independent security audits by qualified cryptographers
- Compliance with industry standards and regulations
- Extensive testing against known attack vectors
- Regular updates and security patches
- Professional support and maintenance

The author(s) make no warranties regarding the security, correctness, or fitness for any particular purpose of this software. Use at your own risk.

## Abstract

PQRSTUV is a cryptographic device used for secure management of cryptographic secrets and with support for Post Quantum Cryptographic algorithms. It does not support symmetric cryptography and is focused on asymmetric cryptography only.

BCP Boilerplate 14:

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119](https://www.rfc-editor.org/rfc/rfc2119.txt) [RFC8174](https://www.rfc-editor.org/rfc/rfc8174.txt) when, and only when, they appear in all capitals, as shown here.

## Technical details
```
                    +----------------------+
                    |    Storage Module    |
                    +----------------------+
                              ||
                              ||  Serial Link (ICP)
                              ||
                    +----------------------+
 [ Network ]--------|   Operation Module   |
             ECP/   +----------------------+
             Ethernet     
```
PQRSTUV is developed as two independent components: Storage Module and Operation Module. The Storage Module is airgapped from the network and has only a serial communication link to the Operation Module. Both modules run custom firmware and do not have a general-purpose operating system.

The system uses two different communication protocols:

- Internal Communication Protocol (ICP): This runs over the serial link and is used only between the Operation Module and Storage Module
- External Communication Protocol (ECP): This runs over network and allows other devices to communicate with the Operation Module.

## Internal Communication Protocol Specification

The storage module and operation modules MUST be connected by a short serial connection, with RS-232, with a baud rate of 9600 bps, on the on-board connectors on the motherboards. The storage module has its own TPM used to hold master keys.

The Storage module MUST NOT have any network card or network stack. The storage module MUST NOT be able to be memory dumped without turning off the module. It MUST NOT have any hibernate or related features that allow memory state to be saved to storage. The TPM MUST be locked with PCR values and MUST NOT be unlocked without the same PCR values. The TPM MUST NOT allow unlocking by other secrets or operations. The storage module MUST implement tamper detection tied to the PCR values. The storage module MUST NOT allow any other form of connectivity than the serial communication running ICP with the operation module. The storage module MUST be shielded in a Faraday-cage-like enclosure/ chassis and MUST have power consumption stabilizers so as to minimize side-channel leakages. Any form of tamper to the chassis or the power stabilizing component MUST trip the PCR values.

The operation module and the storage module communicate with frames of fixed structures as defined in the ICP. The storage module behaves in a request-response manner, in sessions. Except for a few commands, all others are executed only in an authenticated session. This ensures no unauthenticated commands are being executed.

Each frame is defined in the following manner

| Preamble | Payload Length | Payload | Checksum (CRC-32) | Trailer |
| --- | --- | --- | --- | --- |
| 16-byte | 4-byte |     | 4-byte | 16-byte |

The components of the frame described above are:

**Preamble (16-bytes)**: This shows where the frame starts. It is a fixed constant:

`0x00 0x00 0x00 0x00 0x50 0x51 0x52 0x53 0x54 0x55 0x56 0x00 0x00 0x00 0x00 0x00`

**Payload Length (4-bytes)**: This is the length of the payload in bytes. Big-endian encoding is used to convert it to bytes.

**Payload**: The payload is defined in later sections of the specification. It is the commands or their responses.

**Checksum (4-bytes)**: This is the CRC-32 checksum of the Payload Length and Payload section of the frame.

**Trailer (16-bytes)**: This shows where the frame ends. It is a fixed constant:

`0xFF 0xFF 0xFF 0xFF 0x43 0x52 0x59 0x50 0x54 0x41 0x4E 0x45 0xFF 0xFF 0xFF 0xFF`

Further, it is imperative that both modules queue up frames in case there are concurrent ones. While it is generally handled by the kernel of the modules, it is to be considered by independent developers who might be developing the modules from scratch.

It is to be noted that the maximum frame size is 50,000 bytes and any receiving frame greater than that should be dropped. The 50,000 bytes is a generous size limit covering even SLH-DSA-256f signatures which stand at 49,856 bytes.

### Payload:

There are two types of payloads:

- Request payload: This type of payload is sent from the Operation Module to the Storage Module
- Response payload: This type of payload is sent from the Storage Module to the Operation Module.

The two types of payloads are differentiated by the direction of the communication and not by any data flag. They have different structures:

The structure for request payloads is:

| Session | Auth token | Command | Operation Data |
| --- | --- | --- | --- |
| 4-byte | 16-bytes | 1-byte |     |

The structure for response payload is:

| Session | Command | Response code | Response |
| --- | --- | --- | --- |
| 4-byte | 1-byte | 1-byte |     |

The available response codes are:

| **Response** | **Code** |
| --- | --- |
| SUCCESS | `0x00` |
| INVALID_CMD | `0x01` |
| CRYPTO_KEY_MISMATCH | `0x02` |
| INVALID_SYNTAX | `0x03` |
| CHECKSUM_FAIL | `0x04` |
| CMD_REJECTED | `0x05` |
| RATE_LIMITED | `0x06` |
| SESSION_UNAVAILABLE | `0x07` |
| INCORRECT_SECRET | `0x08` |
| CMD_FAIL | `0x09` |
| UNKNOWN_ERR | `0xFF` |

It is to be noted that other than for Response code SUCCESS, the 'Response' field for all other Response packets MUST be empty and have no additional information to prevent information leakage. Further all identical operations, as defined by the same type of operation (decapsulation, signature, hashing, hash comparison, checksum computation, checksum comparison) with same cryptosystem and parameter set MUST be performed in constant time and not leak timing information.

### Request handling:

When the Storage module receives a Request Payload Frame, it identifies the boundaries of the frame by analyzing the preamble and trailer. It then computes checksum over the payload length and payload. \[CRC32(Payload length || Payload)\] and compare the checksum with the one in the request frame. If it does not match, it returns a CHECKSUM_FAIL response. The CHECKSUM_FAIL packet is always sent over session 0xFF 0xFF 0xFF 0xFF and has Command field 0xFF.

If the receiving frame goes over 50,000, the Storage module returns a CMD_REJECTED error. It is always sent over session 0xFF 0xFF 0xFF 0xFF and has command field 0xFF.

Implementation note: The Storage module MAY implement two receiving buffers of size 16-bytes and 50,000 bytes respectively and a receiving flag.

The pseudocode for receiving information is:
```
preamble= 16 byte (const. defined earlier)
trailer= 16 byte (const. defined earlier)
buf1: 16 bytes (queue)
buf2: 50000 bytes (queue)
payload_length: 4 bytes (queue)
recv_flag: boolean = false
ctr: int = 0
receive_data(Incoming_Byte: 1 byte)
{
    if(not recv_flag)
    {
        if(buf1 is full)
        {
            dequeue(buf1, 1 byte)
        }
        enqueue(Incoming_Byte --> buf1)
        if(buf1 == preamble)
        {
            dequeue(buf2, 50000 bytes)
            dequeue(payload_length, 4 bytes)
            recv_flag= true
        }
    }
    else
    {
        if(buf1 is full)
        {
            temp=dequeue(buf1, 1 byte)
        }
        enqueue(Incoming_Byte --> buf1)
        if(buf2 is full or to_integer(payload_length, BIG_ENDIAN)>49960)
        {
            send_error(CMD_REJECTED)
            dequeue(buf2, all)
            dequeue(buf1, all)
            dequeue(payload_length, all)    
            recv_flag=false
        }
        ctr=ctr+1
        enqueue(temp --> buf2)
        if(ctr>=16 && payload_length is not full)
        {
            enqueue(temp --> payload_length)
        }
        if(buf1 == trailer)
        {
            temp=dequeue(buf1, 16 bytes)
            enqueue(temp --> buf2)
            full_frame=dequeue(buf2, all_bytes)
            recv_flag= false
            process_request(full_frame)
        }
    }
}

process_request(full_frame: bytes)
{
    required_info= full_frame.(strip first 16 bytes preamble).(strip last 16 bytes trailer)
    payload_info= full_frame.(strip last 4 bytes)
    checksum=full_frame.(get last 4 bytes)
    if(checksum != CRC32(payload_info))
    {
        send_error(CHECKSUM_FAIL)
    }
    else
    {
        run_command(payload_info) //This function will run the commands described later
    }
}
```
For all commands being executed, the Command field in the response payload MUST be the command code of the respective command.

### Commands:

The available commands for the Storage module are:

| Command | Code |
| --- | --- |
| Device information and session commands | `0x0*` |
| GET_INFO | `0x00` |
| PING | `0x01` |
| INIT | `0x02` |
| Authentication secret management: | `0x1*` |
| SEC_SET_INIT | `0x10` |
| SEC_SET_CONF | `0x11` |
| Reset commands: | `0x2*` |
| DEV_RST | `0x20` |
| CRYPTO_RST | `0x21` |
| Key lifecycle management | `0x3*` |
| KEYGEN | `0x30` |
| KEY_LST | `0x31` |
| KEY_DEL | `0x32` |
| IMPORT | `0x33` |
| GET_PUB | `0x34` |
| Cryptographic operations: | `0x4*` |
| DECAPS | `0x40` |
| SIGN | `0x41` |

The commands that run in an unauthenticated session MUST be run on Session `0x00 0x00 0x00 0x00` and the Auth Token should be `0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00`. The sessions `0x00 0x00 0x00 0x00` and `0xFF 0xFF 0xFF 0xFF` are reserved and MUST NOT be used for authenticated sessions. Any command requiring authenticated sessions sent to these sessions should immediately return error SESSION_UNAVAILABLE in constant time without inspecting the frame further.

Authenticated sessions MUST run only one command only. For more than one command, different sessions are to be used. This prevents leaving authenticated sessions open after the operation is done. However, for multiple sequential operations, application developers are encouraged to use scripts to create sessions sequentially per command. However, applications SHOULD NOT cache/store the user secret for more time than to use the device sequentially for one pre-defined operation. It SHOULD NOT store the user secret while waiting for any process other than waiting for PQRSTUV device. For example, the user may want to run KEYGEN, GET_PUB and SIGN commands sequentially to generate a new keypair, get its public key and sign some data. The application may store the user secret for these 3 only and not while waiting for user input, waiting for any other application or a network call. Applications SHOULD retain user secret only the minimum time required to complete a predefined sequence of operations and MUST NOT persist the secret beyond that. It should be immediately zeroized.

#### Device Information and session commands:

##### GET_INFO (0x00)

GET_INFO command runs in an unauthenticated session. GET_INFO command does not need any Operation data. It returns the device information as CBOR encoded JSON. The information JSON MUST contain a field for 'available_cryptosystems' which is a list containing the COSE Identifiers of the supported algorithms. It MUST contain a field 'token_hash_algo' which is the COSE identifier of the recommended cryptographic hashing algorithm used to generate the Auth tokens. For example, the following JSON shows a GET_INFO output.
```
{
    "name": "PQRSTUV Demo Device",
    "serial_number": "d3f2da78-ca68-4e60-ae6d-b0cc0a503bc8",
    "manufacturer": "Cryptane",
    "documentation": "example.com",
    "available_cryptosystems": [
        -50,
        -49,
        -48,
        -257,
        -7
    ],
    "token_hash_algo": -16
}
```
The GET_INFO command output MUST not change on reboots or by device state. It MAY be cached on the client application.

##### PING (0x01)

PING is used to test the connectivity between the two modules of the device. PING runs in an unauthenticated session. The PING command reads all Operation Data in the request and echoes the same as response.

##### INIT (0x02)

Except for the device information commands and session commands, all other commands require authenticated sessions. Sessions from 0x00 0x00 0x00 0x01 to 0xFF 0xFF 0xFF 0xFE can be randomly allotted.

Each session has three states:

- Unallocated/ Destroyed (UD)
- Allocated, waiting (AW)
- Allocated, processing (AP)

A session set to AW state MUST have a timeout of 10 minutes within which if it does not move to AP state, it will automatically be moved to UD.

INIT runs in an unauthenticated session. It does not require any Operation data. It generates a random session ID which is not in AW or AP states. It further generates a 16-byte Nonce and associates it with the session ID. The session is moved to AW state. The command returns a 20-byte response where the first 4 bytes is the session identifier, and the remaining 16-bytes is the nonce.

| Session Identifier | Nonce |
| --- | --- |
| 4-bytes | 16-bytes |

All other commands require authenticated sessions. To use an authenticated session, the Session field of the command frame MUST contain the session identifier that it had received from the INIT command response. If this is violated, that is if the frame contains a reserved session ID or a session ID in UD state, the error SESSION_UNAVAILABLE is returned without further processing the frame.

The client SHOULD develop the authentication token by concatenating the user secret to the nonce, followed by hashing the same with the algorithm mentioned in 'token_hash_algo' field of the GET_INFO response. The first 16 bytes of the same are used as the authentication token.

`Auth Token = H(Secret || Nonce).truncate(16)`

The Auth Token present in the frame is now validated with the Storage module computing the same. The validation MUST be done in constant time. If the validation fails, the INCORRECT_SECRET error is returned. A violation counter is incremented when the validation fails which is used to implement rate limiting.

The rate limit SHOULD be 3 validation failures in 5 minutes, after which all authenticated requests SHOULD be rate limited for the next 30 minutes. Rate limited requests MUST NOT be parsed, and the RATE_LIMITED error should be returned immediately during this time duration.

#### Authentication secret management commands:

This set of commands is used to change the authentication secret used to generate the authentication token. The secret storage will be described in a later section of 'Secure Storage'. This section assumes the secret is securely stored and can be modified only by these commands.

##### SEC_SET_INIT (0x10):

This command takes 3-byte operation data which contains the COSE identifier of a KEM or Encryption algorithm. The identifier MUST be from the list of 'available_cryptosystems' presented during the GET_INFO call and MUST NOT be a digital signature algorithm. If the identifier is not from the list, the CMD_FAIL error is returned. If it is a digital signature algorithm, the CRYPTO_KEY_MISMATCH error is returned.

The module then generates a keypair of the algorithm referred to by the identifier. This keypair stays in memory and not in storage. The keypair MUST be zeroized after a time-out of 10 minutes. Any other keypairs generated by the same command previously currently in the memory MUST be zeroized immediately.

The response contains the encapsulation key or public key of the newly generated keypair depending on the cryptosystem used. The key is serialized as the CBOR encoding of its COSE format JSON.

##### SEC_SET_CONF (0x11):

The user is expected to enter a new secret on the client device. The secret is encrypted with AES-GCM-256, followed by the AES key being encapsulated or encrypted with the public key/ encapsulation key received in the SEC_SET_INIT call previously. They are appended and is sent as the operation data.

`Operation Data = (Encrypted Secret with AES-GCM-256 || Encapsulated or encrypted AES Key)`

If there was no SEC_SET_INIT call before, or the key was zeroized due to a timeout, the storage module MUST return error CMD_FAIL without further processing of the command.

The AES key is then decapsulated/ decrypted, which is then further used decrypt the user secret. The secret MUST be less than 1024 bytes in size, violation of which will return CMD_FAIL. The user secret is securely stored for future use. A SUCCESS response is returned without any response data. The keypair created during SEC_SET_INIT MUST be zeroized immediately.

#### Reset commands

This set of commands is used to reset certain functionalities of the device

##### DEV_RST (0x20):

All device storage, including the secure storage, the cryptographic storage, user configuration and vendor specific configurations is immediately zeroized and reverted to default configurations. Refer to 'secure storage' section of this document which highlights other cryptographic secrets that MUST be rotated with this command. A SUCCESS response is returned without any response data.

##### CRYPTO_RST (0x21):

All stored cryptographic keys are immediately zeroized and their identifiers are freed. Refer to 'secure storage' section of this document which highlights other cryptographic secrets that MUST be rotated with this command. A SUCCESS response is returned without any response data.

#### Key lifecycle management

This set of commands is used to manage the lifecycle of cryptographic keypairs on the device.

##### KEYGEN (0x30):

This command takes 3-byte operation data which contains the COSE identifier of a cryptographic algorithm. The identifier MUST be from the list of 'available_cryptosystems' presented during the GET_INFO call. If the identifier is not from the list, the CMD_FAIL error is returned.

A new keypair of the given algorithm is generated and a random 16-byte key identifier is generated, which MUST be previously unused. The keypair is saved securely against the 16-byte key identifier and the algorithm identifer. It is immediate zeroized from the memory after storing it in the secure storage.

The command returns the 16-byte identifier in response.

##### KEY_LST (0x31):

This command lists the available keys for a particular cryptosystem. This command takes 3-byte operation data which contains the COSE identifier of a cryptographic algorithm. The identifier MUST be from the list of 'available_cryptosystems' presented during the GET_INFO call. If the identifier is not from the list, the CMD_FAIL error is returned. The storage module MUST NOT load the keypairs into memory from the secure storage while counting. Only the identifiers are to be loaded to memory.

The response contains the count of the number of keypairs for the particular algorithm, in big-endian encoding in the first 4 bytes. The remaining bytes contain 16-byte identifier of each of them, appended in lexicographical order. The order MUST be lexicographical and should not be in be in the order of creation or import, which would leak chronological metadata. Further, it should be easier to sort them in lexicographical order of the identifiers.

##### KEY_DEL (0x32):

This command takes 16-byte operation data which contains the identifier for the key to be deleted. If a key associated with that identifier does not exist, the CMD_FAIL error is returned.

The key associated with the identifier is zeroized, the identifier freed and the command returns SUCCESS without any response data.

##### IMPORT (0x33)

This command takes in CBOR dumped COSE encoding of a keypair. It inspects the algorithm field (1) in the same and if it does not match with any in its own 'available_cryptosystems' list, it MUST immediately return CMD_FAIL and zeroize the input.

A 16-byte random identifier is created, and the private key is stored in the secure storage against the identifier. The 16-byte identifier is returned in response data.

##### GET_PUB (0x34):

This command takes 16-byte operation data which contains the identifier for the key to be deleted. If a key associated with that identifier does not exist, the CMD_FAIL error is returned.

The public key or encapsulating key corresponding to the keypair against the identifier is converted to a COSE dictionary and then CBOR dumped. The keypair is zeroized from the memory after extracting the public key or the encapsulating key and converting it to JSON format. The same is returned in response data.

#### Cryptographic operations

This set of commands is used to perform cryptographic operations with the device

##### DECAPS (0x40):

This command takes the first 16-bytes as the key identifier to be used to decapsulate or decrypt. If there is no keypair corresponding to the identifier in the secure storage, CMD_FAIL is returned. If the keypair corresponding to the identifier is of a Digital signature algorithm, CRYPTO_KEY_MISMATCH is returned.

The remaining operation data contains the ciphertext. If it does not match the ciphertext size of the cryptosystem with parameter set corresponding to the keypair stored against the identifier, CRYPTO_KEY_MISMATCH is returned.

If the ciphertext is decapsulated or decrypted with the private key corresponding to the keypair on secure storage against the identifier in constant time with respect to the algorithm and parameter set. In case of failures, CMD_FAIL should be returned. The plaintext is returned as response data for SUCCESS. The private key or decapsulation key and the plaintext MUST be immediately zeroized after the decapsulation/decryption is done.

##### SIGN (0x41):

This command takes the first 16-bytes as the key identifier to be used to sign. If there is no keypair corresponding to the identifier in the secure storage, CMD_FAIL is returned. If the keypair corresponding to the identifier is of an encapsulation or encryption algorithm, CRYPTO_KEY_MISMATCH is returned.

The remaining operation data contains the cryptographic hash digest of the message to be signed. The HASH should be in SHA3-256

If input size does not match the digest size of the used hashing algorithm, CMD_FAIL should be returned. The digest is now signed with the digital signature algorithm, and the signature is returned as response data of SUCCESS. The private key or signing key MUST be immediately zeroized after the signing is done.

### Secure Storage:

All keypairs MUST be encrypted by AES-GCM-256 before storing it to the standard storage: SSD or Flash. The AES key MUST be generated and stored in a Trusted Platform Module (TPM) and MUST NOT be exportable. The operating system or firmware MUST NOT store any metadata of keypair generation or usage. All cryptographic material MUST immediately be zeroized from memory after using. This AES Key MUST be rotated with the CRYPTO_RST command.

The user secret is also stored in the SSD or Flash after being encrypted with another AES-GCM-256 after padding it to 1024 Bytes. This AES key MUST also be generated and stored in a Trusted Platform Module (TPM) and MUST NOT be exportable.

Both AES Keys in the TPM MUST be rotated with the DEV_RST command.

## External Communication Protocol Specification

The External Communication Protocol (ECP) highlights the communication of the operations module with client and other devices. It is important that the operation module allows connectivity with other devices in order for the device to be usable. It must have a network stack to enable connectivity and works on REST APIs. The operations module MUST run a webserver that allows communication over REST APIs. The operation module MUST NOT allow console access or any other forms of access in wired, wireless, or even physical connection modes. The operation module MUST NOT allow connectivity over any other network ports than the one on which the web server is listening. The operation module MUST have no more than one storage-specific physical connectivity slot (like SD Card or MicroSD card slot) with write-only mechanism and MUST NOT have any general-purpose ports like USB, PCIe etc. The operation module MUST NOT allow any configuration settings and for network configuration it MUST use DHCP and MUST NOT announce itself with mDNS or any other network protocols. The operation module MUST NOT reach out to any servers over the internet and MAY send UDP traffic to a syslog server available on local network. ECP provides specifications on adding a syslog server to operations module. The operation module MUST STRICTLY follow ICP for communication with the storage module over serial link. The operation module SHOULD implement two buffers for the serial data as described in ICP for communicating with the storage module. ECP, wherever mentions Base64 encoding or URL Safe Base64 encoding MUST always be interpreted as URL Safe Base64 encoding without padding. The operation module MAY use asynchronous logic for handling the web server calls but MUST use synchronous command and response queuing for ICP.

The operation module MUST NOT host any static files, debug endpoints, dashboards or any other functionality than the mentioned REST APIs. The operation module MUST have secure connectivity over TLS1.3 or up and the ECP must be executed in secure context only. The operation module TLS certificates will be self-signed due to restricted communication to the internet, however, the ECP provides specifications to export the public key certificate of the TLS which can be added to org-wide CA in enterprise access or to the Certificate Store in client system computers for individual access.

Each ECP API needs an Authorization header as input, which contains the Base64 encoding of the corresponding authorization token as described in ICP.

Request headers:

- Content-Type: application/json
- Session: URL safe Base64 encoded session identifier. It MUST be 'AAAAAA' (Base64 encoding of 0x00 0x00 0x00 0x00) for unauthenticated sessions.
- Authorization: URL safe Base64 encoded authorization token. It MUST be 'AAAAAAAAAAAAAAAAAAAAAA' (Base64 encoding of 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00) for unauthenticated sessions.

Response codes:

- 200: Operation executed
- 400: Input error
- 404: Undefined API
- 403: Authorization header not present or does not encode 16-bytes or Session header not present or does not encode 4-bytes
- 417: Encoding/Decoding failures
- 500: Other errors

The operation module MUST return 200 even if the authorization header is wrong, since the error is handled by the storage module. It is to be noted that in both request and response, if the mentioned data is of type None, it SHOULD be an empty string. The operation module MUST NOT retry ICP commands automatically under any circumstances.

The request structures for POST Requests and response structure for 200 response code are highlighted below. For all other HTTP error codes, the response MUST be empty JSON ({}).

POST Request Structure
```
{
    data: String, JSON or None
}
```
Response structure:
```
{
    code: Big-endian encoding of response code returned by corresponding ICP call. 0 for SUCCESS
    result: String, JSON or None
}
```
The APIs available for ECP are described below:

| Command | Request type | Parameter | Return type |
| --- | --- | --- | --- |
| Device information and session APIs: |     |     |     |
| /info | GET | None | JSON Object |
| /ping | POST | String | String |
| /init | POST | None | JSON Object |
| Authentication secret management: |     |     |     |
| /set_secret | POST | Number | String |
| /confirm_secret | POST | JSON Object | None |
| Reset APIs: |     |     |     |
| /device_reset | POST | None | None |
| /crypto_reset | POST | None | None |
| Key lifecycle management |     |     |     |
| /keygen | POST | Number | String |
| /list_keys | POST | Number | JSON Object |
| /key_delete | POST | String | None |
| /import | POST | String | String |
| /get_public_key | POST | String | String |
| Cryptographic operations: |     |     |     |
| /decapsulate | POST | JSON Object | String |
| /sign | POST | JSON Object | String |
| Operation module management |     |     |     |
| /renew_tls | POST | JSON Object | String |
| /set_syslog | POST | JSON Object | String |

### API Specifications:

All API calls will have an Authorization header as specified previously. The operation module MUST take the Base64 encoded authorization header, decode it to bytes and use it for the corresponding ICP communication. Each API call in ECP has a corresponding ICP call and some additional functionalities. The operation module MUST NOT receive plain text user secret or cache the authorization header.

All POST requests MUST have JSON data in format specified above while GET requests need not have any parameter. POST requests having None as parameter SHOULD use the 'data' field as an empty string. APIs involving reading a 16-byte identifier field from the user in Base64 encoded format MUST ensure that the received identifier is 16-bytes after decoding and if not, immediately return an error code of 417 and stop executing further instructions of the call.

For the corresponding ICP call, if the response code is not 0x00 (SUCCESS), the operation module MUST stop further computation of the call and returns the response with the big-endian encoding of the response code in 'code' and the 'result' field being None.

#### Device information and session APIs

This set of APIs is used to manage the device session states and retrieve information about the device.

##### /info

This API is used to get information about the PQRSTUV device. This is an unauthenticated command. The operation module on receiving this SHOULD send the INFO command to the storage module via ICP. It should decode the response CBOR into JSON Object and it should be the value of the 'data' field in the response.

##### /ping

This API tests the connectivity till the storage module. This is an unauthenticated command. It takes Base64 encoded bytes as data input string, decodes it and makes the PING ICP call with the same data. It re-encodes the response back to Base64 and returns it for response.

##### /init

This API is used to initiate an authenticated session. This is an unauthenticated API. It makes the INIT ICP call. It parses the session identifier and the nonce and encodes them with URL safe Base64 encoding. It makes the response JSON with fields 'session' and 'nonce' with the Base64 strings.

An example response data is:
```
{
    session: "-s76zg",
    nonce: " AAECAwQFBgcICQoLDA0ODw"
}
```
All further APIs are authenticated APIs and require an authenticated session defined by the session and authorization headers.

#### Authentication secret management

This set of APIs is used to manage the user secret used to authorize sessions.

##### /set_secret

This API takes an integer data as an input parameter and converts it to a Big-Endian encoded 3-byte number. It uses this as operation data and makes the SEC_SET_INIT call. It reads the CBOR encoded public key or encapsulation key from the ICP response and converts it to DER format. The DER format is Base64 encoded and returned in the data field of the response.

##### /confirm_secret

This command invokes the SEC_SET_CONF ICP call. Its parameter is a JSON Object with fields 'encrypted_secret' and 'symmetric_key'.

The encrypted_secret is the new user secret, encrypted with AES-GCM-256 and the ciphertext in URL safe Base64 format. The symmetric_key field contains the AES Key encrypted or encapsulated with the public-key or encapsulating key returned by the previous /set_secret call in URL safe Base64 encoding.

An example of request data is:
```
{
    encrypted_secret: URL Safe Base64 Encoded AES Encrypted secret,
    symmetric_key: URL Safe Base64 encoded encrypted/encapsulated AES Key
}
```
The operation module decodes them and makes the SEC_SET_CONF ICP call as per the ICP specifications. It returns a response with None data.

#### Reset APIs

This set of APIs is used to reset certain functionalities of the API.

##### /device_reset

This command invokes the DEV_RST ICP call. It has None type request and responses.

##### /crypto_reset

This command invokes the CRYPTO_RST ICP call. It has None type request and responses.

#### Key lifecycle management

This set of APIs is used to manage lifecycle of cryptographic secrets.

##### /keygen

This API takes an integer data as an input parameter and converts it to a Big-Endian encoded 3-byte number. It uses this as operation data and makes the KEYGEN call. It reads the identifier response from ICP, encodes it to URL Safe Base64 format and returns.

##### /key_delete

This API takes in a 16-byte key identifier encoded as a URL Safe Base64 String and decodes it to bytes. It invokes the ICP command KEY_DEL with the key identifier as parameter. It returns None as response.

##### /import

This command takes in DER encoded keypair in Base64 String as parameter. It converts the same to COSE format and CBOR dumps it. The same is then used to invoke the IMPORT ICP call. The operation module MUST zeroize the received keypair once the ICP invoke is complete. The ICP call returns a 16-byte identifier, which is Base64 encoded and returned as response data string.

##### /get_public_key

This API takes in a 16-byte key identifier encoded as a URL Safe Base64 String and decodes it to bytes. It invokes the ICP command GET_PUB with the key identifier as parameter. It reads the CBOR encoded public key or encapsulation key from the ICP response and converts it to DER format. The DER format is Base64 encoded and returned in the data field of the response.

#### Cryptographic operations

This set of APIs is used to perform cryptographic operations.

##### /decapsulate

This command takes a JSON input with fields 'identifier' and 'ciphertext'. The 'identifier' is the URL Safe Base64 encoding of the key to be used, while 'ciphertext' is the URL Safe Base 64 encoded encapsulated or encrypted secret. An example of the request data is:

```
{
    identifier: URL Safe Base64 Encoded identifier,
    ciphertext: URL Safe Base64 encoded ciphertext
}
```

The operation module decodes them and makes the DECAPS ICP call as per the ICP specifications. It gets the plaintext from the ICP response and encodes it into URL Safe Base64. It returns the same as response data. The operation module MUST immediately zeroize the plaintext after the operation is complete.

##### /sign

This command takes a JSON input with fields 'identifier' and 'document. The 'identifier' is the URL Safe Base64 encoding of the key to be used, while 'document' is the data to be signed. An example of the request data is:
```
{
    identifier: URL Safe Base64 Encoded identifier,
    document: URL Safe Base64 encoded document
}
```

The operation module decodes them and calculates SHA3-256 hash digest of the document. It uses the identifier and the digest and makes the SIGN ICP call as per the ICP specifications. It gets the signature from the ICP response and encodes it into URL Safe Base64. It returns the same as response data.

#### Operation module management

This set of APIs is used to manage the internals of the operation module.

##### /renew_tls

This operation takes in a JSON Object as input containing a field 'export' with a Boolean value, TLS key generation fields as strings and a 16-byte identifier in Base64 encoded format. An example of request data is:

```
{
    export: boolean,
    country_name: String,
    state: String,
    province: String,
    organization_name: String,
    organization_unit_name: String,
    common_name:String,
    identifier: URL Safe Base64 encoded identifier
}
```

The operation module generates a TLS Keypair using the TLS Key generation fields. It generates a certificate signing request from the public key corresponding to the new keypair. It then generates the hash digest of the CSR data. The identifier is decoded to bytes. The identifier and the hash digest are used to make the ICP SIGN call. If the ICP call fails, the request is terminated. The response contains the signature. The signature is appended to the CSR and is used to make the public key certificate in PEM format. If the 'export' boolean is set to 'true', the certificate in CRT and PEM formats is written to the storage media connected to the operation module. The TLS certificates are then rotated to the new certificates. It returns a response with empty data.

The operation module webserver process MUST NOT be involved in generating the TLS Keypair and SHOULD use other vetted applications like OpenSSL to generate the keypair. The webserver process SHOULD open only the public key and CSR. The webserver process MUST NOT communicate with external storage and SHOULD use any operating system process like echo and pipe.

##### /set_syslog

The process to set up a syslog server is a two-step process. The client or user is first expected to create a JSON file containing the syslog server details with fields 'syslog_server_host', 'syslog_port'. An example of the JSON file is:

```
{
    syslog_server_host: 192.168.26.79,
    syslog_port: 514,
}
```

This file is encoded in Base64 format, and the user makes a /sign call to make this an authenticated request.

The /set_syslog call takes a JSON with three parameters, 'syslog_data', 'signature' and 'identifier'. The syslog_data field contains the Base64 encoded JSON file defined at the previous step. The signature field contains the Base64 encoded signature returned in the previous call. The identifier contains the Base64 encoded 16-byte identifier of the key used to sign the request.

The operation module decodes the identifier makes the GET_PUB ICP call and gets the public key corresponding to the keypair with which the request JSON was signed. It then verifies the signature over the 'syslog_data' field with the public key. If the request is valid, the operation module extracts the syslog_server_host and syslog_port values, thus setting it as the new syslog server. If the extraction fails, or the host or port are invalid, the request MUST return a 400 HTTP error code.

### Logging:

If a syslog server is not configured, the operation module MUST NOT log operations since there is no way to extract or analyze the logs. If a syslog server is configured, it SHOULD log all non-0x00 response codes received from ICP with the IP address of the client corresponding to the request. If the syslog server is changed with the /set_syslog call, the operation module MUST report it to the previous syslog server before changing.

All calls of the following categories MUST be logged irrespective of success or failure, including the client IP address from which the request is originating:

- Authentication secret management:
- Key lifecycle management
- Operation module management

The logs MUST never be written to a file on the operation module and MUST instead be sent to the syslog server immediately. The webserver process MUST NOT communicate with the syslog server and MAY use IPC, a sub-process or a call to OS utilities for logging.

Logs MUST NOT contain authorization tokens, session identifiers, cryptographic materials, plaintext secrets or raw request data. Log message formats MUST be stable and MUST NOT include variable-length data derived from the operations. The client IP address used for logging MUST be derived from the TCP connection and MUST NOT be taken from the request headers.

## Deployment guidelines

For secure deployment, the following guidelines SHOULD be followed:

- The REST API Calls over ECP MUST NOT be cached, proxied or be TLS decrypted.
- Proper management of user secret SHOULD be enforced.
- The device SHOULD be deployed in a segmented network and SHOULD NOT be accessible to unauthorized users.
- The device SHOULD have proper physical access management and MUST NOT be physically accessible to unauthorized persons.

## Issues and Contributing

This documentation is available on the repository [AdityaMitra5102/PQRSTUV-Docs](http://github.com/AdityaMitra5102/PQRSTUV-Docs). If any part of this specification appears incorrect, ambiguous, or problematic, please open an issue in the GitHub repository. 


Feedback via direct messages, emails, or informal communication channels will not be tracked and may be missed. All technical discussion and proposed changes should be initiated as GitHub issues or pull requests.

## Acknowledgements

Developed by Cryptane (a.k.a Aditya) [Homepage](https://adityamitra5102.github.io), Prof. Sibi Chakkaravarthy S., in collaboration with Centre of Excellence, Artificial Intelligence and Robotics (AIR), VIT-AP University, Andhra Pradesh, India. [AIR Center](https://air.vitap.ac.in/) and as a R&D project of Digital Fortress Pvt. Ltd. [Digital Fortress](https://digitalfortress.in).
