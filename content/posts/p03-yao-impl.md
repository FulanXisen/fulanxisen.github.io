---
title: "Yao's Protocol Impl in MPCFGO"
date: 2023-04-28T15:45:15+08:00
# weight: 1
# aliases: ["/first"]
tags: ["first"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

# Yao's Protocol Theory


# Impl Yao's Protocol in MPCFGO
## Type Struct
Let's declare a struct to represent a Yao's Protocol variable.
```go
// Yshare The label and pointer are stored in a []byte, [0:len-1]byte, is the label, and [len-1]byte is the pointer
type Yshare struct {
	// Yshare is a share of l bit, Plaintext is a l-bit number
	Plaintext pub.PubNum
	wValue    [][]byte    // wValue[i] is the i-th bit's label and pointer, activated
	wTable    [][2][]byte // wValue[i][0] is the i-th bit's label and pointer of value 0, wValue[i][1] of value 1
}
```
`Yshare` contains three fields:
- `Plaintext`: a `pub.PubNum` type, which represents an l-bit plaintext number.If this Yshare variable is created from garbler's input, this instance in garbler side should own the correct value of Plaintext. If this Yshare variable is create from evaluator's input, this instance in evaluator side should own the correct value of Plaintext. Otherwise, this instance do not own the correct value but own a unset value that has the same type with the correct value.
- `wValue`: a two-dimensional byte slice (`[][]byte`) representing the label and pointer of each bit in the `Plaintext`. Specifically, `wValue[i]` contains the label and pointer of the `i`-th bit of the plaintext. The label and pointer are stored in a `[]byte`, where `[0:len-1]byte` represents the label, and `[len-1]byte` represents the pointer.
- `wTable`: a three-dimensional byte slice (`[][2][]byte`) representing the label and pointer of each possible value (0 or 1) of each bit in the `Plaintext`. Specifically, `wTable[i][j]` contains the label and pointer of the `i`-th bit of the plaintext when it has value `j`. 

The `Yshare` type is used to represent a share of an l-bit secret, where each bit is shared between multiple parties. The `wValue` and `wTable` fields are used to store the shares of each bit and their corresponding labels and pointers.

These are two methods of the `Yshare` type, `New()` and `NewFrom()`, which are used to create new instances of `Yshare` and return them as a `PvtNum` type.

## New, NewFrom
The `New()` method takes a `network.Network` type and a `pub.PubNum` type as inputs, and returns a new `Yshare` instance as a `PvtNum` type. If the input network is the server, it generates the shares for the `Plaintext` using the `newYshare()` function and sends the length of the `Yshare` instance to the client. It then sends the label and pointer for each bit of the `Plaintext` based on the value of each bit. If the input network is not the server, it receives the length of the `Yshare` instance from the server and then receives the label and pointer for each bit of the `Plaintext`.

The `NewFrom()` method takes a `network.Network` type as an input and returns a new `Yshare` instance as a `PvtNum` type. If the input network is the server, it generates a new instance of `pub.PubNum` with a length specified by the length of the `Yshare` instance received from the client. It then sends the label and pointer for each bit of the `Yshare` instance to the client. If the input network is not the server, it receives the `Plaintext` of the `Yshare` instance from the server and then receives the label and pointer for each bit of the `Yshare` instance.

The newYshare function creates a new Yshare object with a specified length.

The wTable field of the Yshare object is initialized as a slice of length length, where each element of the slice is a pair of slices of length config.SymByte+1 bytes. The first slice of each pair is assigned a random byte array of length config.SymByte+1 using the rand_.Read() function. The second slice of each pair is assigned the result of XOR-ing the first slice with a fixed byte array DELTA using the misc.BytesXorBytes() function.

The last byte of each slice in the wTable field is then set to either 0 or 1 based on the result of the randBool() function. If randBool() returns true, the last byte of the first slice is set to 1 and the last byte of the second slice is set to 0. If randBool() returns false, the last byte of the first slice is set to 0 and the last byte of the second slice is set to 1.

Finally, the newYshare function returns the newly created Yshare object.

## Declassify
`decode0` and `decode1` are two methods of the `Yshare` struct that are used to declassify a secret-shared value. Declassification is the process of converting a secret-shared value into a plaintext value that can be used in subsequent computations. 

`decode0` is called by the server to declassify a secret-shared value. It computes the declassified value using the wire values received from the client. The method first computes two hashes, `h0` and `h1`, of the wire values for each share using a pseudorandom function (PRF). It then checks the least significant bit (LSB) of the wire value to determine which hash value to use to compute the corresponding bit of the declassified value. Finally, it sends the declassified value back to the client.

`decode1` is called by the client to declassify a secret-shared value. It computes the declassified value using the wire values received from the server. The method first receives a table `dTable` containing the declassified bits computed by the server. For each share, it computes the LSB of the wire value using a PRF and checks it to determine which bit of `dTable` corresponds to the corresponding bit of the declassified value. Finally, it sends the declassified value back to the server.

## Basic Gate Evaluation
This is an implementation of the "Eval" function in Go. It takes a Bristol circuit, represented by the "bc" parameter, and a set of input Yshares, represented by the "ins" parameter, and evaluates the circuit to produce a single output Yshare.

The "Eval" function performs the evaluation using the provided "net" parameter, which is a network object that represents the communication channels between the parties involved in the computation. If "net.Server" is true, then the current party is the garbler, and the function performs the necessary computations to generate garbled gates and sends them to the other party. If "net.Server" is false, then the current party is the evaluator, and the function waits to receive garbled gates from the garbler, and then evaluates them to produce the final output Yshare.

The implementation uses a set of helper functions to perform the necessary cryptographic operations. For example, the "prf.FixedKeyAES.Hash" function is used to hash keys, and the "misc.BytesXorBytes" function is used to perform XOR operations on byte arrays.

The implementation first performs some sanity checks to ensure that the circuit and input Yshares are valid. It then generates garbled gates for each gate in the circuit using the provided input Yshares. The implementation supports four types of gates: AND, EQ, EQW, NOT, and XOR. For each gate type, the implementation performs the necessary cryptographic operations to generate the corresponding garbled gate, and then sends the garbled gates to the other party over the network.

The implementation stores the garbled gates in the "gWires" array, which is a two-dimensional array of byte arrays. The first dimension corresponds to the output wire of the gate, and the second dimension corresponds to the two possible output values of the gate (0 and 1). The implementation then waits to receive the garbled gate values from the garbler, and uses them to evaluate the garbled circuit to produce the final output Yshare. Finally, the implementation returns the output Yshare.
### XOR
implements an XOR gate using FreeXOR technique. FreeXOR is a technique used to evaluate an XOR gate in a garbled circuit by using only one shared wire instead of two.

The inputs to the XOR gate are the values of the two input wires v.InWire[0] and v.InWire[1]. The values of these wires are retrieved from gWires and stored in the variables lhs and rhs, respectively.

Then, two output wires for the XOR gate are allocated in gWires[v.OutWire][0] and gWires[v.OutWire][1]. The XOR operation is performed on the input wires using misc.BytesXorBytes function and the result is stored in gWires[v.OutWire][0].

Finally, a FreeXOR gate is created using DELTA as the shared wire value and gWires[v.OutWire][0] as the unshared wire value. The result of this FreeXOR operation is stored in gWires[v.OutWire][1].

The result is that gWires[v.OutWire][0] and gWires[v.OutWire][1] now contain the two possible output values of the XOR gate, and these values can be sent to the next gate in the circuit.

### AND
The circuit evaluation used here is based on two primitive gates: the FreeXOR gate and the Halfgate. The FreeXOR gate takes two input wires a and b and computes a XOR b, while the Halfgate takes two input wires a and b and computes a AND b. The Halfgate can be implemented using two FreeXOR gates, as shown in the Halfgates construction.

The code uses the following steps to evaluate an AND gate:

It initializes a lookup table gTable with two entries, each of which contains a byte array of length config.SymByte (the size of the symmetric key used in the circuit).
It retrieves the values of the input wires v.InWire[0] and v.InWire[1] from the gWires array, which stores the values of all wires in the circuit.
It computes the key crKeyP by hashing the value of one of the input wires (either rhs[0] or rhs[1]) using a fixed AES key stored in the prf.FixedKeyAES variable. The choice of input wire to hash depends on the value of lhs[0][config.SymByte], which represents the value of the first bit of the other input wire lhs[0].
If rhs[0][config.SymByte] is zero, it computes the entry gTable[0] by hashing the other value of the second input wire rhs[1] using the same AES key, XORing it with crKeyP, and optionally XORing it with the constant DELTA if lhs[0][config.SymByte] is one. If rhs[0][config.SymByte] is one, it computes the entry gTable[0] using the opposite values of the input wires.
It computes the key clKeyP by hashing the value of the input wire lhs[0] using the same fixed AES key. The choice of input wire to hash depends on the value of r, which represents the value of the first bit of the output wire v.OutWire.
It computes the entry gTable[1] by hashing the other value of the input wire lhs[1] using the same AES key, XORing it with clKeyP and rhs[0], and optionally XORing it with DELTA if r is one.
It computes the values of the output wire v.OutWire by XORing clKeyP and crKeyP and storing the result in gWires[v.OutWire][0], and by XORing this result with DELTA and storing the result in gWires[v.OutWire][1].
It sends the two entries of the lookup table gTable[0] and gTable[1] to the other party over the network using the net.Send function.
The code assumes that the gWires array contains the correct values for all wires in the circuit, and that the prf.FixedKeyAES variable contains a fixed AES key that is shared between the two parties. 

### XOR