---
title: "Algoritmo de Assinatura Digital com Curvas Elípticas (ECDSA)"
---

# Algoritmo de Assinatura Digital com Curvas Elípticas (ECDSA)

ECDSA é um algoritmo de assinatura digital que utiliza criptografia de curva elíptica. Uma chave privada é usada para assinar uma mensagem, e uma chave pública é usada para verificar a assinatura.

A mensagem passa por um hash (como SHA-256) antes de ser assinada.

```go
import (
	"crypto/ecdsa"
	"crypto/rand"
	"crypto/sha256"
)

msg := "Olá mundo!"
hash := sha256.Sum256([]byte(msg))
signature, err := ecdsa.SignASN1(rand.Reader, privateKey, hash[:])
```

## Assinaturas

Assinaturas ECDSA são representadas usando um par de inteiros positivos, (r, s).

### IEEE P1363

No formato IEEE P1363, a assinatura é a concatenação de r e s. Os valores são codificados como bytes em big-endian com tamanho equivalente ao da curva utilizada. Por exemplo, no P-256, cada valor tem 256 bits (ou 32 bytes).

```ts
r || s;
```

### PKIX

Na [RFC 5480](https://datatracker.ietf.org/doc/html/rfc5480) do grupo de trabalho PKIX, a assinatura é codificada em ASN.1 DER como uma sequência de r e s.

```
SEQUENCE {
    r     INTEGER,
    s     INTEGER
}
```

## Chaves Públicas

Chaves públicas ECDSA são representadas como um par de inteiros positivos, (x, y).

### SEC1

No padrão [SEC 1](https://www.secg.org/sec1-v2.pdf), as chaves públicas podem ser codificadas em formato não comprimido (uncompressed) ou comprimido (compressed).

Chaves não comprimidas são a concatenação de x e y, com um byte `0x04` no início. Os valores são codificados como bytes [big-endian](<https://pt.wikipedia.org/wiki/Extremidade_(ordena%C3%A7%C3%A3o)>) com um tamanho equivalente ao tamanho da curva. Por exemplo, P-256 tem 256 bits (ou 32 bytes) de tamanho.

```
0x04 || x || y
```

Chaves comprimidas são o valor de x com um byte inicial 0x02 quando x for par ou um byte inicial 0x03 quando x for ímpar. O valor de y pode ser derivado de x e da curva.

```
0x02 || x
0x03 || x
```

### PKIX

Na (https://datatracker.ietf.org/doc/html/rfc5480 do grupo de trabalho PKIX, a chave pública é representada como uma sequência ASN.1 `SubjectPublicKeyInfo`. A `subjectPublicKey` pode ser a chave pública SEC1 comprimida ou não comprimida.

```
SubjectPublicKeyInfo := SEQUENCE {
    algorithm           AlgorithmIdentifier,
    subjectPublicKey    BIT STRING
}
```

O `AlgorithmIdentifier` para ECDSA é uma sequência ASN.1 com o identificador de objeto ECDSA (`1.2.840.10045.2.1`) e a curva (por exemplo, `1.2.840.10045.3.1.7` para a curva P-256)

```
AlgorithmIdentifier := SEQUENCE {
    algorithm   OBJECT IDENTIFIER
    namedCurve  OBJECT IDENTIFIER
}
```
