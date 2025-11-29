+++
title = "Protocol Buffers Encoding"
date = 2025-11-29
+++

If you've ever worked with proto, you might be aware of best practices such as "Don’t Re-use a Tag Number" or "Do Reserve Tag Numbers for Deleted Fields". You might also have questions such as "Is it safe to rename a field?" or "How does proto ensure backward compatiblity of clients when the server updates its contract?". These questions can be answered once we understand how proto sends messages over the wire.

The most fun way I can think of learning this is to spin up a basic gRPC client and server locally ([guide](https://grpc.io/docs/languages/go/quickstart/)), and analysing the network packets using Wireshark. 


# Example 1:
Let's begin with the following proto definition of the request:

```
message HelloRequest {
  string name = 1;
}
```

This is the Wireshark output for the request from the gRPC client to the server:

![wireshark](/proto-wireshark/0.png)

Since we are interested in understanding how the proto message is serialized, let's focus on that. 

```
0a 0b 52 61 76 69 20 53 61 6e 6b 65 72
```

[52 61 76 69 20 53 61 6e 6b 65 72] clearly maps to "Ravi Sanker", which is the value of the "name" field in the request. 0b is 11 in decimal which is the number of characters in "Ravi Sanker". Since these are all ASCII characters, UTF-8 encoding takes only a byte for each character. So 11 stands for 11 bytes. But what is 0a? Wireshark is telling us it's for something called the "Field Number" and "Wire Type". It's time to refer to the [docs](https://protobuf.dev/programming-guides/encoding/).

According to the docs, each field is serialized into a tag and value. tag is a combination of the field number and the type. The exact formula is:
```
tag := (field << 3) bit-or wire_type [encoded as uint32 varint]
```
In our example, the field number is 1 and the type is string which falls under the "LEN" wire type which has the ID 2. So using the formula, we get:
```
0000 1XXX
0000 0010
---------
0000 1010
---------
```
00001010b is 0xA. So this all adds up. 

**The important point to note here is that proto does not care about the field name.**

# Example 2:
Let's update our request to:
```
message HelloRequest {
  string name = 1;

  reserved 2;

  int32 number = 16;
}
```

If the client were to pass `name="Ravi Sanker"` and `number=100`, what do you think the serialization would be? Well, since the tag and value for name hasn't changed, I would expect it to be the same as the previous example. For number, since its field number is 16 and the docs say int32 falls under the "VARINT" wire type which has the ID 0, I would expect this field to be 0x80 0x64. 0x80 is the tag and 0x64 is the value.

```
1000 0XXX
0000 0000
---------
1000 0000
---------
```

Let's take a look at what Wireshark is saying:

![wireshark](/proto-wireshark/1.png)

```
0a 0b 52 61 76 69 20 53 61 6e 6b 65 72 80 01 64
```
[0a 0b 52 61 76 69 20 53 61 6e 6b 65 72] is expected. But what's the extra 01 doing in [80 01 64]? Do integers have a length field as well? That doesn't make any sense.

The answer lies in the subtle "encoded as uint32 varint" statement of the formula. Maybe the encoding of 128 is not 128. From the docs, this is how the encoding for uint32 works:
```
       1  0000000 (original integer)
 0000001  0000000 (split the integer into 7 bits)
 0000000  0000001 (convert to little endian)
10000000 00000001 (add continuation bit)
0x80     0x01
```
Now it adds up. But why is the encoding of 100 (0x64) the same? 100 is 0b1100100 which is strictly less than a byte. The varint encoding of such numbers is the number itself.

Why is this so complicated? An obvious benefit is that we are only using as many bytes as necessary. All integers don't take up the same space. Storing in little endian format improves efficiency, though I don't completely understand why.

# Conclusion

Now coming back to the questions we had, yes, it's perfectly safe to rename fields as proto only cares about the field number and the type. In fact, it's safe to even change the type in some cases. It's not recommended to reuse tag numbers (`protoc` compiler actually returns an error) as each field is uniquely identified with this number. It's recommended to reserve deleted fields so that fields introduced in the future don't use the same field number and a client that's using the old contract doesn't break. 

Regarding backward compatibility of clients: 

- If the client is on a later version of the contract while the server isn't, the server will simply drop the field numbers that it doesn't recognise.
    - If you're wondering how it's even possible that the client is on a newer version, this can happen if service A that has a client to service B gets deployed to a higher environment with the latest contract before service B is deployed.
- If the client is on an older version, assuming the newer contract has followed the best practices of not updating the field number of an existing field and not updating the type of an existing field to an incompatible type, proto guarantees that the client will be compatible.

# Bonus

Accoring to the [FAQ](https://grpc.io/docs/what-is-grpc/faq/#what-does-grpc-stand-for), gRPC stands for gRPC Remote Procedure Calls. So yet another recursive definition of an acronym. But each [release](https://github.com/grpc/grpc/blob/master/doc/g_stands_for.md) of gRPC defines `g` to be something different.
