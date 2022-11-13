---
layout: post
title:  "Solidity ABI argument encoding with Perl"
date:   2022-10-24 01:47:28 -0300
categories: solidity
---
This is an introduction to the [perl-ABI](https://github.com/refeco/perl-ABI) repo, you can find it through the following channels:
- [Metacpan](https://metacpan.org/pod/REFECO::Blockchain::Contract::Solidity::ABI)
- [Github](https://github.com/refeco/perl-ABI)

Before proceed, I would recommend you to read the [contract ABI specification](https://docs.soliditylang.org/en/v0.8.17/abi-spec.html)

This repo has two main modules: `Encoder` and `Decoder`, let's give a try on some real world transactions to understand how they are intended to be used.

While using e.g. `eth_call` or `eth_sendTransaction` you can use the field `data` to interact with contracts, the data in this field must be encoded following the [contract ABI specification](https://docs.soliditylang.org/en/v0.8.17/abi-spec.html) (if you are interacting with contracts), and this is exactly what this module does.

# Table of contents

- [Encoding](#encoding)
- [Decoding](#decoding)
- [Using it](#using-it)

## Encoding

**ERC20 transfer call:**

- [transaction](https://etherscan.io/tx/0x9a3425ab9eab399ecb51560fadf98350b6eb9e27881daffb113dc6acfecbca3d)
- function signature: `transfer(address _to, uint256 _value)`

Example:

{% highlight perl %}
my $address = '0x89000a6ce8B62d7F949B4D394FACc35B35Afe8A7';
# the module does not handle the decimal places,
# the formatting needs to take place before the call
my $value   = sprintf "%18.f", 351.50 * 1E6;
my $encoded = $encoder->function('transfer')
                ->append(address => $address)
                ->append(uint256 => $value)
                ->encode;
say $encoded;
# that is no new line in the response, I've added it for better visualization
# 0xa9059cbb
# 00000000000000000000000089000a6ce8B62d7F949B4D394FACc35B35Afe8A7
# 0000000000000000000000000000000000000000000000000000000014f376e0
{% endhighlight %}

In this sample both arguments are static, so that is no need for indexes or size information, basically what is happening here is:

 - `0xa9059cbb` - The first 4 bytes (without 0x) are related to the function signature: first four bytes from keccak256(signature)
 - After this point the stack is parsed by 32 bytes
 - `00000000000000000000000089000a6ce8B62d7F949B4D394FACc35B35Afe8A7` address
 - `0000000000000000000000000000000000000000000000000000000014f376e0` value

This is a very simple example and you would not face any issues doing it without any module, but let's try something more challenging:

**Seaport fulfillBasicOrder call:**

- [transaction](https://etherscan.io/tx/0x5f38348937b9fedd3e2463c47eb6575c8f9189cbbbec70e4eec66fabdac7394f)
- function signature: `fulfillBasicOrder(tuple parameters)`

Example:

{% highlight perl %}
my $encoded = $encoder->function('fulfillBasicOrder')->append(
    '(address,uint256,uint256,address,address,address,uint256,uint256,uint8,uint256,uint256,bytes32,uint256,bytes32,bytes32,uint256,(uint256,address)[],bytes)'
        => [
        '0x0000000000000000000000000000000000000000',
        0,
        2850000000000000,
        '0x5E9988B0C47B47A5b1d7D2E65358789044C2eF9a',
        '0x004C00500000aD104D7DBd00e3ae0A5C00560C00',
        '0x2Cc8342d7c8BFf5A213eb2cdE39DE9a59b3461A7',
        45981, 1, 2,
        1664873641,
        1667544153,
        '0x0000000000000000000000000000000000000000000000000000000000000000',
        '24446860302761739304752683030156737591518664810215442929812187193154675427388',
        '0x0000007b02230091a7ed01230072f7006a004d60a8d4e71d599b8104250f0000',
        '0x0000007b02230091a7ed01230072f7006a004d60a8d4e71d599b8104250f0000',
        3,
        [
            [75000000000000, '0x0000a26b00c1F0DF003000390027140000fAa719'],
            [30000000000000, '0x2Dc05E282a6829c66e91b655F91129800Fb9DBDF'],
            [45000000000000, '0x2a1De3F4582BB617225e32922Ba789693156FC8C']
        ],
        '0xd410c95bb9d3f04cb0c2340a5fa890e506c2d9159aadd8bf06bcd03e56a9f48c44115a13a2f9fc6184cfe40747ebff2da100329aa385fa19de112ed2f3ed11731b'
        ])->encode;
say $encoded;
# that is no new line in the response, I've added it for better visualization
# 0xfb0f3ee1
# 0000000000000000000000000000000000000000000000000000000000000020
# 0000000000000000000000000000000000000000000000000000000000000000
# 0000000000000000000000000000000000000000000000000000000000000000
# 000000000000000000000000000000000000000000000000000a200f559c2000
# 0000000000000000000000005e9988b0c47b47a5b1d7d2e65358789044c2ef9a
# 000000000000000000000000004c00500000ad104d7dbd00e3ae0a5c00560c00
# 0000000000000000000000002cc8342d7c8bff5a213eb2cde39de9a59b3461a7
# 000000000000000000000000000000000000000000000000000000000000b39d
# ... the result is long so I've just truncated it, you can find the
# full response bellow in the breakdown
{% endhighlight %}

As you can see, this one would be harder to handle even if you need to do this only once, let's break down the information related to this transaction:

First as you can see etherscan shows this function signature as `fulfillBasicOrder(tuple parameters)` but this is only for better visualization, not everyone cares about this information, but that is no `tuple` type being interpreted here, basically when you give an `struct` as parameter for a solidity function, in the signature, all the fields from the `struct` must be informed, so for sample:

for the following solidity function:

{% highlight javascript %}
struct Test {
    uint256 a;
    address b;
}

function runTest (Test calldata t) public view{}
{% endhighlight %}

The signature for the function `runTest` is `runTest((uint256,address))`, the extra parentheses are to identify the argument as a tuple.

Now the encoded response:

- `0xfb0f3ee1` - The first 4 bytes (without 0x) are related to the function signature: first four bytes from keccak256(signature)
- `0000000000000000000000000000000000000000000000000000000000000020` dynamic tuple index \
**... skipping a bunch of static fields here since we have seen how they work in the previous example**
- `0000000000000000000000000000000000000000000000000000000000000240` dynamic tuple index
- `0000000000000000000000000000000000000000000000000000000000000320` dynamic bytes index
- `0000000000000000000000000000000000000000000000000000000000000003` tuple array size
- `000000000000000000000000000000000000000000000000000044364c5bb000` static uint
- `0000000000000000000000000000a26b00c1f0df003000390027140000faa719` static address
- `00000000000000000000000000000000000000000000000000001b48eb57e000` static uint
- `0000000000000000000000002dc05e282a6829c66e91b655f91129800fb9dbdf` static address
- `000000000000000000000000000000000000000000000000000028ed6103d000` static uint
- `0000000000000000000000002a1de3f4582bb617225e32922ba789693156fc8c` static address
- `0000000000000000000000000000000000000000000000000000000000000041` bytes size
- `d410c95bb9d3f04cb0c2340a5fa890e506c2d9159aadd8bf06bcd03e56a9f48c` the dynamic bytes value has more than 32 bytes so it is break in 3 chunks of 32 bytes
- `44115a13a2f9fc6184cfe40747ebff2da100329aa385fa19de112ed2f3ed1173`
- `1b00000000000000000000000000000000000000000000000000000000000000`

Bytes and strings are padded right while everything else is padded left.

# Decoding

For decoding the logic is the same as the encoding, the difference from the Encoder module is that for decoding, you append only the type signatures to the module and pass the value to the decode function as the following example using the same call as the last encoding sample (without the function name):

{% highlight perl %}
my $decoded = $decoder->append(
        '(address,uint256,uint256,address,address,address,uint256,uint256,uint8,uint256,uint256,bytes32,uint256,bytes32,bytes32,uint256,(uint256,address)[],bytes)'
    )->decode($data);
say $decoded;
# [
#   '0x0000000000000000000000000000000000000000',
#   bless( {
#     '_a' => undef,
#     '_p' => undef,
#     'sign' => '+',
#     'value' => bless( [
#       0
#     ], 'Math::BigInt::Calc' )
#   }, 'Math::BigInt' ),
#   bless( {
#     '_a' => undef,
#     '_p' => undef,
#     'sign' => '+',
#     'value' => bless( [
#       0,
#       2850000
#     ], 'Math::BigInt::Calc' )
#   }, 'Math::BigInt' ),
#   '0x5e9988b0c47b47a5b1d7d2e65358789044c2ef9a',
#   '0x004c00500000ad104d7dbd00e3ae0a5c00560c00',
#   '0x2cc8342d7c8bff5a213eb2cde39de9a59b3461a7',
# ... the result is long so I've just truncated it
{% endhighlight %}

Is also worth mention that the numeric values are returned always as Math::BigInt instances, and the encoder module also accepts Math::BigInt as input for the numeric type signatures.

## Using it

That is a real sample of how you can use both encode and decoding modules to abstract a contract call, we are using the `balanceOf` function, common in many token contracts, this function receive an address as argument and returns an uint256 as response, following the working sample with the http call to infura (you need to have the infura key, or change the endpoint to your local node instead).

{% highlight perl %}
#!/usr/bin/env perl

use strict;
use warnings;
use feature qw/say/;

use LWP::UserAgent;
use JSON;

use REFECO::Blockchain::Contract::Solidity::ABI::Encoder;
use REFECO::Blockchain::Contract::Solidity::ABI::Decoder;

my $endpoint = 'https://mainnet.infura.io/v3/YOUR_KEY_HERE';
my $ua       = LWP::UserAgent->new;

my $encoder = REFECO::Blockchain::Contract::Solidity::ABI::Encoder->new;
my $decoder = REFECO::Blockchain::Contract::Solidity::ABI::Decoder->new;

my $contract = '0xdAC17F958D2ee523a2206206994597C13D831ec7';

sub request {
    my ($method, $params) = @_;
    my $request = HTTP::Request->new('POST', $endpoint);
    $request->header('Content-Type' => 'application/json');

    my $data = {
        id     => 1,
        method => $method,
        params => [$params]};

    $request->content(encode_json({id => 1, method => $method, params => $params}));
    return $ua->request($request)->decoded_content;
}

sub balanceOf {
    my $address  = shift;
    my $encoded  = $encoder->function('balanceOf')->append(address => $address)->encode;
    my $response = request(
        'eth_call',
        [{
                to   => $contract,
                data => $encoded
            },
            'latest'
        ]);

    return $decoder->append('uint256')->decode(decode_json($response)->{result});
}

say balanceOf('0x5041ed759Dd4aFc3a72b8192C143F72f4724081A')->[0]->bstr;
{% endhighlight %}

Hopefully this has been helpful for anyone in need of a library to do that, if you have any questions or requests, feel free to open an github issue.

See ya!
