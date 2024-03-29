# HYDRA

![GitHub](https://img.shields.io/github/license/goealabs/dotnet-crypto-hydra?style=for-the-badge)
![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/goealabs/dotnet-crypto-hydra?include_prereleases&style=for-the-badge)
![Nuget (with prereleases)](https://img.shields.io/nuget/vpre/GoeaLabs.Crypto.Hydra?style=for-the-badge)

# Project description

This is the reference implementation of HYDRA, an authenticated stream cipher. 
HYDRA draws its lineage from the ChaCha family of ciphers, which presently includes ChaCha20, XChaCha20 and 
XChaCha20-Poly1305. 

While it shares this heritage, HYDRA stands apart from its predecessors with a distinct design and features. Although it 
may have originated from the same cryptographic family tree, it has been evolved to offer unique capabilities and 
enhancements beyond what the ChaCha ciphers provide.

The main characteristics of ChaCha20, XChaCha20-Poly1305 and HYDRA at a glance:

| Cipher Name        | Key Size (bits) | Nonce Size (bits) | Rounds     | Hashing Algorithm(s)                                                            | Encrypts output signature | Secret key can encrypt max (bytes) |
|--------------------|-----------------|-------------------|------------|---------------------------------------------------------------------------------|---------------------------|------------------------------------|
| ChaCha20           | 256             | 96                | 20         | N/A                                                                             | N/A                       | 2<sup>32</sup>                     |
| XChaCha20          | 256             | 192               | 20         | N/A                                                                             | No                        | 2<sup>64</sup>                     |
| XChaCha20-Poly1305 | 256             | 192               | 20         | Poly1305                                                                        | No                        | 2<sup>64</sup>                     |
| HYDRA              | 256             | 256               | &#8805; 20 | SHA256<sup>1</sup>, SHA384<sup>1</sup>, SHA512<sup>1</sup> & custom<sup>2</sup> | Yes                       | 2<sup>389 <sup>3</sup>             |

<sup>1</sup> HYDRA encrypts the resulting ciphertext signature as well and its security does not depend on HMAC with 
separate secret key. For that reason, the built-in algorithms are limited to the SHA2 family of functions.
<br>
<sup>2</sup> Users can plug-in any additional hashing algorithms, including algorithms requiring a separate secret key,
such as HMAC-SHA(256|384|512), Poly1305 etc.
<br>
<sup>3</sup> 2<sup>256</sup> * 2<sup>64</sup> * 2<sup>6</sup> * 2<sup>63</sup> = 2<sup>389</sup>.

# Technical summary

The key to understanding HYDRA lies in understanding its key system (pun intended). During the encryption process, 
HYDRA makes use of either 4 or 5 keys, depending on whether the hashing algorithm of choice requires a secret key of
its own.

The following table displays an overview of these keys and their purpose:

| Key Name | Key Size (bits) | Generated by | Managed by | Description                                                                                                        |
|----------|-----------------|--------------|------------|--------------------------------------------------------------------------------------------------------------------|
| X-KEY    | 256             | User         | User       | The secret key. It is never <sup>1</sup> used to encrypt data directly.                                            |
| N-KEY    | 256             | HYDRA        | HYDRA      | The nonce. Randomly generated every time encryption is performed.                                                  |
| E-KEY    | 256             | HYDRA        | HYDRA      | The encryption key (E-KEY = X-KEY &#x2295; N-KEY)                                                                  |
| H-KEY    | Variable        | HYDRA        | HYDRA      | Hashing key if a particular hashing algorithm requires one. Randomly generated every time encryption is performed. |
| S-KEY    | Variable        | HYDRA        | HYDRA      | Signature encryption key. Randomly generated every time encryption is performed.                                   |

<sup>1</sup> The probability that the X-KEY directly participates in encryption is 1/2<sup>256</sup>, because there is a 
probability of 1/2<sup>256</sup> of randomly generating an all zero random N-KEY, in which case E-KEY and X-KEY will 
be the same. This has no impact on the security of the cipher.

Beyond its unique key system, HYDRA has the following attributes:

- It has 3 built-in hashing schemes and allows consumers to plug-in their own algorithms;
- It not only signs the resulting ciphertext, but it also encrypts the resulting signature, thus enabling the use of 
hashing algorithms that do not require a secret key. The reason why this might be desirable is that usually keyed 
hashing algorithms need to perform multiple passes over the data which negatively impacts performance.
- Although up to 5 keys participate in the encryption/decryption process, the user is never burdened with the management 
of any key beyond the secret key (X-KEY);

# Library Features

- Apache 2.0 license;
- Supports all .NET platforms, including WebAssembly;
- Fully managed;
- No unsafe code;
- Simple API;

## Usage example

```csharp
using GoeaLabs.Crypto.Hydra;

// Use an existing X-KEY
const string xorKey = "982F7130FB1C675B06A42765D2AD64C38E8F156271BE6143617A0899F57A5257";

// Alternatively, generate a new random X-KEY as HEX string

//var xorKey = HydraEngine.NewKey();

// Alternatively, generate a new random X-KEY as byte array

//var xorKey = new byte[HydraEngine.KeyLen];
//HydraEngine.NewKey(xKey);

// Use plain SHA256 signatures
var signer = new Sha256Signer();

// Or any of the other built-in algorithms:

//var signer = new Sha384Signer();
//var signer = new Sha512Signer();

// Initialize Hydra with our signer and default rounds (20)
var engine = new HydraEngine(xorKey, signer);

// Alternatively, initialize Hydra with our signer and custom number of rounds

//var engine = new HydraEngine(xKey, signer, 100);

// Write the encryption the current encryption scheme:
Console.WriteLine($"Engine scheme: {engine.Scheme}");

Console.WriteLine($"Engine X-KEY : {xorKey}");

var plaintext = "This is a plaintext message. It is also very secret!"u8.ToArray();

Console.WriteLine($"Plaintext HEX: {Convert.ToHexString(plaintext)}");

// Encrypt the data
Span<byte> encrypted = stackalloc byte[engine.GetLen(plaintext, isPlain: true)];
engine.Encrypt(plaintext, encrypted);

Console.WriteLine($"Encrypted HEX: {Convert.ToHexString(encrypted)}");

// Decrypt the data
Span<byte> decrypted = stackalloc byte[engine.GetLen(encrypted, isPlain: false)];
engine.Decrypt(encrypted, decrypted);

Console.WriteLine($"Decrypted HEX: {Convert.ToHexString(decrypted)}");
```

## Plugging-in custom signers

Plugging-in a new signer is as simple as implementing `IHydraSigner` interface, eg:

```csharp
using GoeaLabs.Crypto.Hydra;

namespace MyCompany.MyLibrary;

public class MyCustomAlgo : IHydraSigner
{
    // Define the hashing scheme name:
    
    /// <inheritdoc/>
    public string Scheme => "MY_ALGO"; 
    
    // Define your algorithms' key length in bytes. If it doesn't need one, set it to 0:
    
    /// <inheritdoc/>
    public int KeyLen => 0;

    // Define your algorithms' signature size in bytes. Eg, 16 bytes (128 bit):
    
    /// <inheritdoc/>
    public int SigLen => 16;

    /// <inheritdoc/>
    public void GenSig(ReadOnlySpan<byte> src, ReadOnlySpan<byte> key, Span<byte> sig)
    {        
        // Hash the data in 'src' and write the output in 'sig'. 'key' is automatically 
        // populated by Hydra with 'KeyLen' cryptographically secure random bytes if 'KeyLen' > 0. 
    }
}
```

## Built-in signer performance

The following benchmarks have been performed on a development machine. They are only meant to compare the performance of
the built-in `IHydraSigner` implementations relative to each other and not as an indicator of absolute encryption
performance.


`IHydraSigner` encryption performance for 100, 1000, and 1000000 bytes with 20 rounds.

```
BenchmarkDotNet v0.13.12, Ubuntu 23.10 (Mantic Minotaur)
Intel Core i7-9750H CPU 2.60GHz, 1 CPU, 12 logical and 6 physical cores
.NET SDK 8.0.102
  [Host]     : .NET 8.0.2 (8.0.224.6711), X64 RyuJIT AVX2
  DefaultJob : .NET 8.0.2 (8.0.224.6711), X64 RyuJIT AVX2
```

| Method               | Bytes   |          Mean |      Error |     StdDev | Rank | Allocated |
|----------------------|---------|--------------:|-----------:|-----------:|-----:|----------:|
| Hydra20Sha256Encrypt | 100     |      3.794 μs |  0.0083 μs |  0.0077 μs |    1 |         - |
| Hydra20Sha384Encrypt | 100     |      3.850 μs |  0.0093 μs |  0.0082 μs |    2 |         - |
| Hydra20Sha512Encrypt | 100     |      3.909 μs |  0.0089 μs |  0.0083 μs |    3 |         - |
| Hydra20Sha384Encrypt | 1000    |     14.381 μs |  0.0201 μs |  0.0188 μs |    4 |         - |
| Hydra20Sha512Encrypt | 1000    |     14.685 μs |  0.0240 μs |  0.0213 μs |    5 |         - |
| Hydra20Sha256Encrypt | 1000    |     15.074 μs |  0.0220 μs |  0.0206 μs |    6 |         - |
| Hydra20Sha512Encrypt | 1000000 | 11,211.651 μs | 40.3984 μs | 35.8121 μs |    7 |      12 B |
| Hydra20Sha384Encrypt | 1000000 | 11,769.714 μs | 16.2679 μs | 15.2170 μs |    8 |      12 B |
| Hydra20Sha256Encrypt | 1000000 | 12,478.181 μs | 11.2396 μs | 10.5135 μs |    9 |      12 B |


`IHydraSigner` decryption performance for 100, 1000, and 1000000 bytes with 20 rounds.

```
BenchmarkDotNet v0.13.12, Ubuntu 23.10 (Mantic Minotaur)
Intel Core i7-9750H CPU 2.60GHz, 1 CPU, 12 logical and 6 physical cores
.NET SDK 8.0.102
  [Host]     : .NET 8.0.2 (8.0.224.6711), X64 RyuJIT AVX2
  DefaultJob : .NET 8.0.2 (8.0.224.6711), X64 RyuJIT AVX2

```

| Method               | Bytes   |          Mean |      Error |     StdDev | Rank | Allocated |
|----------------------|---------|--------------:|-----------:|-----------:|-----:|----------:|
| Hydra20Sha384Decrypt | 100     |      2.823 μs |  0.0137 μs |  0.0121 μs |    1 |         - |
| Hydra20Sha512Decrypt | 100     |      2.892 μs |  0.0124 μs |  0.0116 μs |    2 |         - |
| Hydra20Sha256Decrypt | 100     |      2.930 μs |  0.0161 μs |  0.0151 μs |    3 |         - |
| Hydra20Sha512Decrypt | 1000    |     13.072 μs |  0.0344 μs |  0.0322 μs |    4 |         - |
| Hydra20Sha384Decrypt | 1000    |     13.480 μs |  0.1239 μs |  0.1159 μs |    5 |         - |
| Hydra20Sha256Decrypt | 1000    |     13.502 μs |  0.0214 μs |  0.0190 μs |    5 |         - |
| Hydra20Sha512Decrypt | 1000000 | 11,327.440 μs | 36.1948 μs | 32.0858 μs |    6 |      12 B |
| Hydra20Sha384Decrypt | 1000000 | 11,815.813 μs | 14.6395 μs | 12.2247 μs |    7 |      12 B |
| Hydra20Sha256Decrypt | 1000000 | 12,305.134 μs | 36.1246 μs | 28.2037 μs |    8 |      12 B |


## Installation

Install with NuGet Package Manager Console
```
Install-Package GoeaLabs.Crypto.Hydra
```

Install with .NET CLI
```
dotnet add package GoeaLabs.Crypto.Hydra
```
