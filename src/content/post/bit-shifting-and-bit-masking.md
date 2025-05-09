---
title: "Bit Manipulation and Protocol Parsing: DNS Server"
publishDate: "9 May 2025"
description: "A refresher on bit masking and shifiting with DNS Server Message Header"
tags: ["DNS", "Bits", "Golang", "Protocol"]
---

## Bit Shifting and Bit Masking - A Refresher on Bit Manipulation with DNS Message Header

I recently took on the Codecrafters DNS Server Challenge where I built a DNS server from scratch. I’ve always been fascinated by how foundational technologies work under the hood, and this was the perfect opportunity to dive deep. Building this, I picked up some programming skills specifically bit shifting, bit masking, and understanding endianness.

In this article, I will walk you through what I learned. These techniques are crucial when decoding and encoding DNS message headers, which are streams of bytes that require manipulation to correctly represent the DNS protocol.

### What is Bit Shifting?

Bit shifting involves moving bits to the left or right in a binary number, based on a specified shift amount. It's a fundamental operation in low-level programming and useful in manipulating DNS message headers.

##### Example: Shifting Left (<<)
If we have the number `10` (which is `00001010` in binary) and we shift it left by 2:

- **1st shift**: `00010100`
- **2nd shift**: `00101000`

This results in `40` in decimal (or `0x28` in hexadecimal).

##### Example: Shifting Right (>>)
If we shift `10` (binary `00001010`) right by 2:

- **1st shift**: `00000101`
- **2nd shift**: `00000010`

This results in `2` in decimal (or `0x02` in hexadecimal).

###### Summary:
- Shifting **left** (`<<`) multiplies the value by `2^n` (where `n` is the number of shifts).
- Shifting **right** (`>>`) divides the value by `2^n`.

### What is Bit Masking?

Bit masking is used to manipulate a sequence of bits in order to extract or store specific information. This is done using bitwise operations like **AND**, **OR**, and **XOR**. We can mask specific bits to retain or clear parts of a binary number.

##### Example: Extracting Specific Bits
Consider the byte `10010110`. If we want to get the last 3 bits, we can use a mask `00000111` (which is `0x07` in hexadecimal). By performing a **bitwise AND** operation:

```text
10010110
& 00000111
-----------
  00000110
```

##### Common Bitwise Operations

- **AND (`&`)**: Extract bits
- **OR (`|`)**: Set bits
- **XOR (`^`)**: Toggle bits

### So, What Does This Have to Do with a DNS Server?

DNS messages are sent over UDP as tightly packed byte streams. These streams often contain multiple fields jammed into a single byte (for example, flags), and decoding or encoding them properly requires a solid understanding of both bit masking and bit shifting.

Let’s take the DNS header as an example. It’s exactly 12 bytes long and looks something like this

### DNS Header Breakdown (12 bytes)

| Field                          | Size       | Description                                                                 |
|---------------------------------|------------|-----------------------------------------------------------------------------|
| ID                              | 16 bits    | Unique ID for matching queries and responses                                 |
| QR (Query/Response)             | 1 bit      | 1 for response, 0 for query                                                  |
| OPCODE                          | 4 bits     | Type of query                                                                |
| AA (Authoritative)              | 1 bit      | If this server is authoritative                                              |
| TC (Truncated)                  | 1 bit      | If the message was truncated                                                 |
| RD (Recursion Desired)          | 1 bit      | Whether recursion is desired                                                 |
| RA (Recursion Avail.)           | 1 bit      | Whether recursion is supported                                               |
| Z (Reserved)                    | 3 bits     | Reserved bits                                                                |
| RCODE                           | 4 bits     | Response code                                                                |
| QDCOUNT                         | 16 bits    | Question count                                                               |
| ANCOUNT                         | 16 bits    | Answer count                                                                 |
| NSCOUNT                         | 16 bits    | Authority record count                                                       |
| ARCOUNT                         | 16 bits    | Additional record count                                                      |

## Decoding the Header in Go

Here’s a Go function that decodes the first 12 bytes of the DNS message:

```go
func decodeHeaderFromBuffer(buf []byte) Header {
    headerBytes := buf[:12]
    header := Header{}

    header.id = uint16(headerBytes[0])<<8 | uint16(headerBytes[1])

    flags1 := headerBytes[2]
    flags2 := headerBytes[3]

    header.queryResponse = flags1>>7 > 0
    header.operationCode = (flags1 >> 3) & 0x0F
    header.recursionDesired = flags1&1 != 0
    header.truncatedMessage = flags1&(1<<1) != 0
    header.authoritativeAnswer = flags1&(1<<2) != 0
    header.recursionAvailable = flags2&(1<<7) != 0
    header.reserved = (flags2 >> 4) & 0x07
    header.responseCode = RCODE(flags2 & 0x0F)

    header.questionCount = uint16(headerBytes[4])<<8 | uint16(headerBytes[5])
    header.answerCount = uint16(headerBytes[6])<<8 | uint16(headerBytes[7])
    header.authorityCount = uint16(headerBytes[8])<<8 | uint16(headerBytes[9])
    header.additionalCount = uint16(headerBytes[10])<<8 | uint16(headerBytes[11])

    return header
}
```

Let's walk through decoding the DNS header using bit shifting and masking

#### 1. ID (Number in big endian format) - 16 bits
1. The ID is a 16-bit field that is stored in big-endian format. We can extract this by combining two bytes

Let’s walk through converting two bytes `uint8{69, 79}` or `{0x45, 0x4F}` in hex into a 16-bit number.

##### Convert to binary:

- `0x45` = `01000101`
- `0x4F` = `01001111`

Since DNS uses **big endian**, the first byte is the most significant. Therefore, the most significant byte (`0x45`) comes first, followed by the least significant byte (`0x4F`).

To get the full 16-bit value:

```go
uint16(0x45)<<8 | uint16(0x4F)
```

- Shift the first byte (`0x45`) to the left by 8 bits:  
   `0x45 << 8 = 01000101 00000000`

- Now, OR the result with the second byte (`0x4F`):  
   `01000101 00000000 | 01001111 = 01000101 01001111`

That gives `01000101_01001111 (binary) = 17743 (decimal)`

#### 2. Query/Response Indicator (QR) - 1 bit
The QR field is the first bit in the byte, indicating whether the message is a query (0) or a response (1).

**Extraction method:** Shift the byte 7 bits to the right.

**Example:**
```
Original byte: 1000_1000
Operation:     1000_1000 >> 7 = 0000_0001
Result:        1 (Response)
```

#### 3. Operation Code (OPCODE) - 4 bits
The OPCODE field represents the type of DNS message and occupies bits 2-5.

**Extraction method:** Shift right by 3 bits, then mask with 0x0F (0000_1111).

**Example:**
```
Original byte: 1011_0010
Shift by 3:    1011_0010 >> 3 = 0001_0110
Mask with 0x0F: 0001_0110 & 0000_1111 = 0000_0110
Result:        6 (Standard Query)
```

#### 4. Authoritative Answer (AA) - 1 bit
The AA field is the 6th bit in the byte, indicating if the responding server is authoritative.

**Extraction method:** Shift right by 2 bits, then mask with 0x01 (0000_0001).

**Example:**
```
Original byte: 0001_0110
Shift by 2:    0001_0110 >> 2 = 0000_0101
Mask with 0x01: 0000_0101 & 0000_0001 = 0000_0001
Result:        1 (True - server is authoritative)
```

#### 5. Truncation (TC) - 1 bit
The TC field is the 7th bit in the byte, indicating if the message is truncated.

**Extraction method:** Shift right by 1 bit, then mask with 0x01 (0000_0001).

**Example:**
```
Original byte: 0001_0110
Shift by 1:    0001_0110 >> 1 = 0000_1011
Mask with 0x01: 0000_1011 & 0000_0001 = 0000_0001
Result:        1 (Message is truncated)
```

#### 6. Recursion Desired (RD) - 1 bit
The RD field is the least significant bit of the first byte, indicating if the client wants recursive queries.

**Extraction method:** Mask directly with 0x01 (0000_0001).

**Example:**
```
Original byte: 0001_0110
Mask with 0x01: 0001_0110 & 0000_0001 = 0000_0000
Result:        0 (Recursion not desired)
```

#### 7. Recursion Available (RA) - 1 bit
The RA field is the most significant bit of the second byte, indicating if the server supports recursion.

**Extraction method:** Shift the second byte right by 7 bits.

**Example:**
```
Original byte: 1010_0000
Shift by 7:    1010_0000 >> 7 = 0000_0001
Result:        1 (Recursion available)
```

#### 8. Reserved (Z) - 3 bits
The Reserved field is a 3-bit field reserved for future use, occupying bits 5-7 of the second byte.

**Extraction method:** Shift right by 4 bits, then mask with 0x07 (0000_0111).

**Example:**
```
Original byte: 1010_0000
Shift by 4:    1010_0000 >> 4 = 0000_1010
Mask with 0x07: 0000_1010 & 0000_0111 = 0000_0010
Result:        2 (Reserved field value)
```

#### 9. Response Code (RCODE) - 4 bits
The RCODE field occupies the last 4 bits of the second byte, indicating response status.

**Extraction method:** Mask with 0x0F (0000_1111).

**Example:**
```
Original byte: 1010_0000
Mask with 0x0F: 1010_0000 & 0000_1111 = 0000_0000
Result:        0 (No error - successful response)
```

As you can see the Query Response, Operation Required, Recursion Desired, Truncated Message and Authoritative Answer are all gotten from one byte(the first flag byte), while Recursion Available, Reserved and Response Code are gotten from the second byte. It's common practice especially in low-level programming to combine multiple flags or values into a single byte using bitwise operations. This technique is both space-efficient and performant. 

A familiar example of this is Linux file permissions, where read, write, and execute permissions are encoded using a 3-bit representation for each user category (owner, group, others). These are then expressed as octal numbers, such as 755 or 644, making it easy to manage permissions with concise notation while still maintaining fine-grained control.
See a representative diagram below
![File Mode Binary](https://qph.cf2.quoracdn.net/main-qimg-e88bfc7dcc0f29f9123c4cc70ad48da5)


## Encoding the DNS Header

Now that we've walked through decoding a DNS header, let's explore how to encode it back into bytes from the structured data in a Header struct. Here's the function responsible for encoding

```go
func (header *Header) encode(buf *bytes.Buffer) {
	headerBytes := make([]byte, 12)
	headerBytes[0] = byte(header.id >> 8)
	headerBytes[1] = byte(header.id)
	headerBytes[2] = byte(BoolToInt(header.queryResponse)<<7 | header.operationCode<<3 | BoolToInt(header.authoritativeAnswer)<<2 | BoolToInt(header.truncatedMessage)<<1 | BoolToInt(header.recursionDesired)<<0)
	headerBytes[3] = byte(BoolToInt(header.recursionAvailable)<<7 | header.reserved<<3 | uint8(header.responseCode))
	binary.BigEndian.PutUint16(headerBytes[4:6], header.questionCount)
	binary.BigEndian.PutUint16(headerBytes[6:8], header.answerCount)
	binary.BigEndian.PutUint16(headerBytes[8:10], header.authorityCount)
	binary.BigEndian.PutUint16(headerBytes[10:], header.additionalCount)

	buf.Write(headerBytes)
}
```

We now reconstruct the header according to the DNS specification using the booleans and integers available in the Header struct.

### Encoding a 16-bit Integer in Big Endian for the ID

Take an example: you want to encode the integer 17743 into 2 bytes using big endian.

#### Step 1: Convert to Binary

17743 in binary is:
`01000101_01001111`

We want the most significant byte first (big endian). To isolate the first 8 bits (MSB), we shift the value right by 8 bits and apply a mask

```go
byte1 := uint8(17743 >> 8)     // 0x45
byte2 := uint8(17743 & 0xFF)   // 0x4F
```

```
17743 >> 8 = 0x45 = 01000101

17743 & 0xFF = 0x4F = 01001111
```

So, the resulting byte slice becomes:
`[]byte{0x45, 0x4F}`

In the code, we use:

```go
headerBytes[0] = byte(header.id >> 8)
headerBytes[1] = byte(header.id)
```

This performs the same masking and shifting implicitly the `byte()` casts the binary to the last 8 bits which is what we get when we mask with `0xFF`.

### Encoding the Flags Byte

The third and fourth bytes (`headerBytes[2]` and `headerBytes[3]`) store a packed combination of multiple flags and values. Here's how we encode them

### Boolean to Integer Helper

```go
func BoolToInt(b bool) uint8 {
	if b {
		return 1
	}
	return 0
}
```

This converts a boolean value to 0 or 1, which we can use in bitwise operations.

### Constructing the Third Byte

We combine the fields using left-shifts and bitwise OR operations:

```go
headerBytes[2] = byte(
	BoolToInt(header.queryResponse)<<7 |
	header.operationCode<<3 |
	BoolToInt(header.authoritativeAnswer)<<2 |
	BoolToInt(header.truncatedMessage)<<1 |
	BoolToInt(header.recursionDesired),
)
```

**Example:**

Let's assume the following values:

- queryResponse = true -> 1 << 7 = `10000000`
- operationCode = 9 -> `1001` << 3 = `01001000`
- authoritativeAnswer = false -> 0 << 2 = `00000000`
- truncatedMessage = true -> 1 << 1 = `00000010`
- recursionDesired = true  1 << 0 = `00000001`

Now combining all with bitwise OR:

```
10000000
| 01001000
| 00000000
| 00000010
| 00000001
= 11001011
```

Final value: `11001011` (binary) -> `0xCB` (hex)  -> `203` (Uint8)

So `headerBytes[2] = 0xCB`

### Constructing the Fourth Byte

Similar to the third byte, this packs three values:

```go
headerBytes[3] = byte(
	BoolToInt(header.recursionAvailable)<<7 |
	header.reserved<<3 |
	uint8(header.responseCode),
)
```

Each field is shifted to its correct position before combining. For example, if:

- recursionAvailable = true -> 1 << 7 = `10000000`
- reserved = 2 -> 2 << 3 = `00010000`
- responseCode = 0 -> `00000000`

Combining them:

```
10000000
| 00010000
| 00000000
= 10010000
```

So `headerBytes[3] = 0x90` -> `144` (Uint8)

### Encoding Remaining 16-bit Fields

For each of the final header fields (questionCount, answerCount, authorityCount, additionalCount), we store them as 2 bytes (16 bits) in big endian using the Go standard library:

```go
binary.BigEndian.PutUint16(headerBytes[4:6], header.questionCount)
binary.BigEndian.PutUint16(headerBytes[6:8], header.answerCount)
binary.BigEndian.PutUint16(headerBytes[8:10], header.authorityCount)
binary.BigEndian.PutUint16(headerBytes[10:], header.additionalCount)
```

This abstracts away the need to shift and mask manually, as `binary.BigEndian.PutUint16()` handles that for us.

### Conclusion

Working with binary protocols like DNS gives you a hands-on refresher in some of the most fundamental techniques in computer science: bit masking, bit shifting, and understanding how data is laid out in memory. It's a beautiful intersection of theory and practicality where getting the details right really matters.

Check out this interesting post by [Andrew Healey](https://healeycodes.com/visualizing-chess-bitboards) on visualizing chess with bitboard and see how bit manipulation can be very efficient