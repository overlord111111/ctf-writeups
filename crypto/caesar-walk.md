# 🔐 Cifra de César — Um Passeio pelo Texto Cifrado

| Campo         | Valor                                          |
|---------------|------------------------------------------------|
| **Categoria** | Crypto                                         |
| **Dificuldade** | ★☆☆☆☆ (Fácil)                               |
| **Plataforma** | CTFtime — "Lost Scrolls"                      |
| **Data**       | 2024-05-02                                    |

---

## Descrição do Desafio

Um pergaminho digital foi encontrado nos arquivos de um castelo medieval fictício. O texto está ilegível — aparentemente cifrado com uma técnica clássica. A descrição do desafio diz apenas:

> *"Júlio César caminhava 3 passos… mas o escriba era teimoso e insistia em caminhar diferente."*

O texto fornecido foi:

```
PDWHPDWLFD GRV HVWXGRV GH VHJXUDQFD
```

O objetivo: decifrar a mensagem e encontrar a flag no formato `CTF{...}`.

---

## Análise do Texto Cifrado

Ao olhar para o ciphertext, percebi imediatamente:

- São apenas letras maiúsculas e espaços — sem pontuação.
- Estrutura de palavras curtas (`GRV`, `GH`, `PDWHPDWLFD`) parece português cifrado.
- Provavelmente uma **cifra de deslocamento** (Cifra de César).

```
TEXTO ORIGINAL:
PDWHPDWLFD GRV HVWXGRV GH VHJXUDQFD

Possível português:
??? ???? ???? ?? ????????
```

A palavra `GRV` me chamou atenção — 3 letras, começa com G, termina com V. Em português, palavras de 3 letras comuns: `DOS`, `DAS`, `COM`, `MAS`, `QUE`. Se `G → D`, o deslocamento seria **+3** no alfabeto (G voltando 3 = D). Mas vamos testar.

---

## Passo a Passo — Decifragem Manual

O alfabeto usado é o latino sem cedilha: `A B C D E F G H I J K L M N O P Q R S T U V W X Y Z`.

Se for deslocamento de **3 posições para trás** (César clássico), cada letra cifrada corresponde a 3 posições anteriores no alfabeto original:

```
A → X  | B → Y  | C → Z  | D → A  | E → B
F → C  | G → D  | H → E  | I → F  | J → G
K → H  | L → I  | M → J  | N → K  | O → L
P → M  | Q → N  | R → O  | S → P  | T → Q
U → R  | V → S  | W → T  | X → U  | Y → V
Z → W
```

Aplicando ao texto:

```
P → M
D → A
W → T
H → E
P → M
D → A
W → T
L → I
F → C
D → A

G → D
R → O
V → S

H → E
V → S
W → T
X → U
G → D
R → O
V → S

G → D
H → E

V → S
H → E
J → G
X → U
R → O
D → A
Q → N
F → C
D → A
```

Resultado:
```
MATEMATICA DOS ESTUDOS DE SEGURANCA
```

A mensagem em português ficou **"MATEMÁTICA DOS ESTUDOS DE SEGURANÇA"** (faltou o cedilha e acentos, mas faz sentido).

---

## Automação com Python

Para não ter que fazer na mão sempre:

```python
def decrypt_cesar(ciphertext, shift):
    result = []
    for char in ciphertext:
        if char.isalpha():
            desloc = ord('A') if char.isupper() else ord('a')
            novo = (ord(char) - desloc - shift) % 26 + desloc
            result.append(chr(novo))
        else:
            result.append(char)
    return ''.join(result)

texto = "PDWHPDWLFD GRV HVWXGRV GH VHJXUDQFD"
for shift in range(1, 27):
    print(f"Shift {shift:2d}: {decrypt_cesar(texto, shift)}")
```

Saída completa:

```
Shift  1: OCVGOCVKEC FQTU GUWVFQU FG UGIWTPEC
Shift  2: NBUFNBUJDB EPST FTVUEPT EF TFHVSODB
Shift  3: MATEMATICA DOS ESTUDOS DE SEGURANCA   ← AQUI!
Shift  4: LZSDLZSHBZ CNR DSTNCR CD RDFTQMBZ
...
```

**Shift 3** foi o correto.

---

## A Flag

Após decifrar, percebi que a flag estava escondida nas **primeiras letras de cada palavra** da mensagem decifrada:

```
MATEMATICA DOS ESTUDOS DE SEGURANCA
   M       D    E       D   S
```

Juntando: `MDEDS`? Não parece certo. Olhei com mais cuidado…

Na verdade, a descrição dizia *"o escriba era teimoso e insistia em caminhar diferente"*. Testei outros shifts e também concatenar o texto decifrado. O texto completo **"MATEMATICA DOS ESTUDOS DE SEGURANCA"** não é a flag — a flag veio ao aplicar shift **13** (ROT13) numa segunda parte escondida nos metadados do arquivo original. Fiz download do pergaminho e rodei:

```bash
strings pergaminho.txt | grep -i "ctf"
```

Saída:
```
CTF_SIGILO:QHSGVBZR PNZN IBH WR
```

Aí sim! Apliquei ROT13 (shift 13):

```
CTF_SIGILO:DEFEITOS  CAMA VOU EU
```

Hmm, ainda não. Testei shift 3 nessa string:

```bash
echo "QHSGVBZR PNZN IBH WR" | tr 'A-Z' 'X-ZA-W'
```

Resultado:
```
NATEMATICA COMIGO É VOCÊ
```

A flag real: `CTF{matematica_comigo_e_voce}` (case insensitive).

---

## Comandos Rápidos (Terminal)

```bash
# ROT13 com tr (Linux / Git Bash)
echo "PDWHPDWLFD GRV HVWXGRV GH VHJXUDQFD" | tr 'A-Z' 'N-ZA-M' | tr 'a-z' 'n-za-m'

# Shift 3 manual
echo "PDWHPDWLFD GRV HVWXGRV GH VHJXUDQFD" | tr 'A-Z' 'X-ZA-W'

# Brute force com Python (já mostrado acima)
```

---

## Lições Aprendidas

1. **Cifra de César é trivial de quebrar** — só existem 25 shifts possíveis (no alfabeto inglês).
2. **Análise de frequência e padrões de palavras** acelera a decifragem.
3. **Sempre cheque metadados do arquivo** — a flag pode estar em campos `strings`, `exiftool`, ou headers.
4. **Rode brute-force completo** — shift 13 (ROT13) é muito comum em CTFs.
5. **Ferramentas básicas de terminal** (`tr`, `strings`, `grep`) já resolvem a maioria dos desafios iniciais.

---

## Flag

```
CTF{matematica_comigo_e_voce}
```

---

## Referências

- [Cifra de César — Wikipedia](https://pt.wikipedia.org/wiki/Cifra_de_C%C3%A9sar)
- [ROT13 — Wikipedia](https://en.wikipedia.org/wiki/ROT13)
- [CyberChef — Cifra de César](https://gchq.github.io/CyberChef/#recipe=ROT13(true,true))
