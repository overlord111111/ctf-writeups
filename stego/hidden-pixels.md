# 🖼️ Esteganografia — Pixels Ocultos

| Campo         | Valor                                          |
|---------------|------------------------------------------------|
| **Categoria** | Esteganografia                                 |
| **Dificuldade** | ★★☆☆☆ (Médio)                              |
| **Plataforma** | TryHackMe — Room "PixelSecret"                |
| **Data**       | 2024-07-22                                    |

---

## Descrição do Desafio

Uma foto de um gato foi postada num fórum com a legenda *"Será que só o que os olhos veem?"*. Há rumores de que a imagem contém uma mensagem escondida — a tal flag.

A imagem: `cat_secret.png` (300 × 300 pixels, 72 dpi).

```bash
$ file cat_secret.png
cat_secret.png: PNG Image, 300 x 300, 8-bit/color RGBA, non-interlaced
```

```
  ╔══════════════════════════════════╗
  ║       ┌─────────────────┐        ║
  ║       │   🐱😺😸😹😻    │        ║
  ║       │  GATO FELINO    │        ║
  ║       │  (300 x 300)    │        ║
  ║       └─────────────────┘        ║
  ║                                   ║
  ║   "Será que só o que os olhos    ║
  ║    veem?"                         ║
  ╚══════════════════════════════════╝
```

---

## Enumeração

### 1. Metadados — `exiftool`

```bash
$ exiftool cat_secret.png
```

Saída resumida:
```
ExifTool Version Number         : 12.70
File Name                       : cat_secret.png
File Size                       : 85 kB
Image Width                     : 300
Image Height                    : 300
Bits Per Sample                 : 8
Color Type                      : RGB with Alpha
Comment                         : Nada aqui...
Author                          : Anonymous
Copyright                       : TryHackMe 2024
```

O campo `Comment` diz "Nada aqui..." — suspeito. Talvez isca.

### 2. Strings ocultas

```bash
$ strings cat_secret.png | grep -i "ctf\|flag\|htb\|thm\|hidden"
```

Saída:
```
THM{??}
```

Apenas um placeholder. Olhando no meio do binário:

```bash
$ strings cat_secret.png | tail -20
IEND
```

Nada de interessante em texto puro. A mensagem deve estar codificada nos próprios pixels.

---

## Análise Técnica — LSB Steganography

PNG com alpha (RGBA) armazena 4 bytes por pixel (R, G, B, A). Cada byte tem 8 bits. A técnica **LSB (Least Significant Bit)** substitui o bit menos significativo de cada canal por um bit da mensagem oculta. A diferença visual é imperceptível.

```
Pixel original:    R=1101010[0]  G=1011001[1]  B=0110101[0]  A=1111000[0]
Mensagem bits:         1            0            1            1
Pixel modificado:  R=1101010[1]  G=1011001[0]  B=0110101[1]  A=1111000[1]
                          ↑             ↑             ↑             ↑
                     apenas o último bit mudou
```

A variação de cor é de no máximo 1 num canal 0-255 — invisível ao olho humano.

---

## Extração — Passo a Passo

### Ferramenta 1: `zsteg` (a mais fácil)

```bash
$ zsteg cat_secret.png
```

Saída:
```
b1,rgb,lsb,xy       :: text: "THM{st3g0_lsb_1s_fun}"
b2,r,msb,xy         :: text: "Nada aqui..."
```

**Primeira camada (LSB RGB):** `"THM{st3g0_lsb_1s_fun}"`

A flag já apareceu! Mas vamos confirmar com análise manual para aprendizado.

### Ferramenta 2: `stegsolve` (GUI Java)

Abri a imagem no `stegsolve.jar` e fui alternando os planos de bits:

```
Navigate → Browse → cat_secret.png
Analyse → Data Extract
  Planes: Red 0, Green 0, Blue 0 (LSB de cada)
  Bit Order: LSB First
  Bit Plane Order: RGB
```

No preview, apareceu a mesma flag.

### Ferramenta 3: Extração manual com Python

```python
from PIL import Image

img = Image.open("cat_secret.png")
pixels = list(img.getdata())

bits = []
for r, g, b, a in pixels:
    bits.append(str(r & 1))  # LSB do Red
    bits.append(str(g & 1))  # LSB do Green
    bits.append(str(b & 1))  # LSB do Blue
    # Ignorando Alpha porque zsteg mostrou b1,rgb

# Converte bits em bytes e bytes em texto
chars = []
for i in range(0, len(bits), 8):
    byte = int("".join(bits[i:i+8]), 2)
    if 32 <= byte <= 126:  # caracteres imprimíveis
        chars.append(chr(byte))
    else:
        break

print("".join(chars))
```

Saída:
```
THM{st3g0_lsb_1s_fun}
```

A extração manual confirmou o resultado do `zsteg`.

---

## E se o LSB não der resultado?

Sempre bom testar outras camadas e planos de bits:

```bash
# Analisar todas as combinações possíveis
zsteg -a cat_secret.png

# Testar bit 2 em vez de LSB
zsteg -e b2,rgb,lsb,xy cat_secret.png

# Verificar se há dados no canal Alpha
zsteg --all cat_secret.png | grep -i "alpha"
```

Para esteganografia mais sofisticada, ferramentas adicionais:

```bash
# steghide (JPEG, BMP, WAV)
steghide extract -sf cat_secret.png -p ""

# binwalk para arquivos embutidos
binwalk cat_secret.png

# foremost para carving
foremost cat_secret.png -o output/
```

---

## Screenshot ASCII — Extração Visual

```
  ┌─────────────────────────────────────┐
  │  zsteg cat_secret.png              │
  ├─────────────────────────────────────┤
  │                                     │
  │  b1,rgb,lsb,xy  : "THM{st3g0_lsb_  │
  │                   1s_fun}"          │
  │                                     │
  │  b2,r,msb,xy    : "Nada aqui..."   │
  │                                     │
  └─────────────────────────────────────┘

  Aqui vemos duas camadas:
  1. LSB nos canais RGB → Flag real
  2. MSB no canal R, bit 2 → Isca (falso positivo)
  O criador colocou um texto falso em outro
  plano de bits para despistar!
```

---

## Lições Aprendidas

1. **`strings` e `exiftool` são o ponto de partida** — mas em estego, a mensagem raramente está em texto puro.
2. **LSB é a técnica mais comum em CTF** — esconde bits nos canais de cor sem alterar a aparência.
3. **`zsteg` é a ferramenta rei para PNG** — detecta automaticamente LSB, MSB, e combinações de planos de bits.
4. **Sempre cheque múltiplas camadas** — o criador pode colocar iscas em planos diferentes para enganar.
5. **Extração manual em Python** ajuda a entender o que está acontecendo nos bastidores.
6. **Ferramentas não substituem compreensão** — saber como LSB funciona te permite diagnosticar quando as ferramentas falham.

---

## Bônus — Prevenção

Se você fosse criar um desafio de estego:

```python
from PIL import Image
import binascii

def embed_lsb(texto, img_path, output_path):
    img = Image.open(img_path)
    pixels = list(img.getdata())
    bits = ''.join(format(ord(c), '08b') for c in texto) + '00000000'  # null terminator

    new_pixels = []
    for i, (r, g, b, a) in enumerate(pixels):
        if i * 3 < len(bits):
            r = (r & 0xFE) | int(bits[i*3], 2)     if i*3 < len(bits) else r
        if i * 3 + 1 < len(bits):
            g = (g & 0xFE) | int(bits[i*3+1], 2)   if i*3+1 < len(bits) else g
        if i * 3 + 2 < len(bits):
            b = (b & 0xFE) | int(bits[i*3+2], 2)   if i*3+2 < len(bits) else b
        new_pixels.append((r, g, b, a))

    img.putdata(new_pixels)
    img.save(output_path)
    print(f"Mensagem embutida em {output_path}")
```

---

## Flag

```
THM{st3g0_lsb_1s_fun}
```

---

## Referências

- [zsteg no GitHub](https://github.com/zed-0xff/zsteg)
- [StegSolve](https://github.com/eugenekolo/sec-tools/tree/master/stego/stegsolve)
- [LSB Steganography — Wikipedia](https://en.wikipedia.org/wiki/Least_significant_bit)
- [CTF 101 — Steganography](https://ctf101.org/forensics/what-is-stegano/)
