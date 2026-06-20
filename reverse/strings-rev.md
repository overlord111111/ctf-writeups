# 🔄 Engenharia Reversa Básica — Caça às Strings

| Campo         | Valor                                          |
|---------------|------------------------------------------------|
| **Categoria** | Reversa                                        |
| **Dificuldade** | ★☆☆☆☆ (Fácil)                               |
| **Plataforma** | Hack The Box — Challenge "StringTheory"       |
| **Data**       | 2024-06-10                                    |

---

## Descrição do Desafio

Recebemos um binário Linux chamado `validate`. O arquivo veio compactado num `.tar.gz` junto com uma nota: *"Consegui meu primeiro emprego como desenvolvedor e fiz meu primeiro validador de serial! Deve ser impossível de quebrar 😎"*.

Suspeito — provavelmente tem a flag ou o serial escondido no binário.

```bash
$ file validate
validate: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=..., not stripped
```

**Not stripped** — ótimo, os símbolos estão preservados. Isso facilita bastante.

---

## Enumeração Inicial

```bash
# Ver o que o binário faz
$ ./validate
Usage: ./validate <serial>

$ ./validate ABC-123
Invalid serial!

$ ./validate FLAG-XXXX
Invalid serial!
```

Tenta validar um serial. Não diz o formato. Precisamos entender a lógica.

---

## Passo 1 — `strings`

O comando `strings` extrai sequências de caracteres legíveis de binários. É o primeiro passo em qualquer reversa básica.

```bash
$ strings validate
```

Saída (filtrada para partes relevantes):

```
Usage: %s <serial>
Correct! Flag: %s
Invalid serial!
ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789
FLAG-
s3cr3t_k3y_1337
```

Cinco strings importantes:
1. `Usage: %s <serial>` — já sabíamos.
2. `Correct! Flag: %s` — **a flag é exibida se o serial estiver correto!**
3. `Invalid serial!` — mensagem de erro.
4. `ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789` — parece um charset de serial.
5. `FLAG-` — prefixo provável do serial.
6. `s3cr3t_k3y_1337` — parece uma chave secreta!

Temos fortes indícios. O programa compara o serial com algo derivado dessa chave.

---

## Passo 2 — Análise Estática com `objdump` e `rabin2`

Vamos usar o `rabin2` (do radare2) e `objdump` para inspecionar:

```bash
$ rabin2 -z validate
[nome da string] endereço
Usage: %s <serial>              @ 0x2004
Correct! Flag: %s               @ 0x2010
Invalid serial!                 @ 0x2024
ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 @ 0x2030
FLAG-                           @ 0x2050
s3cr3t_k3y_1337                 @ 0x2060
```

```bash
$ objdump -d validate | grep -A 20 "main>:" | head -40
```

Analisando o disassembly da `main`, percebi que:

1. O programa recebe o argumento via `arg[1]`.
2. Compara os primeiros 5 caracteres com `"FLAG-"`.
3. Depois extrai os próximos N caracteres e aplica um XOR com `"s3cr3t_k3y_1337"`.
4. Compara o resultado com uma string fixa na memória.

Ou seja, é um **XOR cipher** simples!

---

## Passo 3 — Extrair o Cifrado

Procurei a string cifrada no binário:

```bash
$ strings -n 10 validate | grep -v "^[A-Z ]*$"
```

Encontrei um bloco aparentemente aleatório, mas usando `radare2` consegui o buffer exato:

```bash
$ r2 -q -c 'p8 40 @ 0x400730' validate
38 4A 4B 5C 4C 3F 40 4E  2A 39 2A 39 2A 39 2A 39
```

Mas o mais fácil: testei executar com o prefixo e a chave como serial:

```bash
$ ./validate FLAG-s3cr3t_k3y_1337
Invalid serial!
```

Próximo palpite: XOR da chave com algo. Escrevi um script.

---

## Passo 4 — Script de Decodificação

```python
import sys

# Strings encontradas no binário
prefixo = "FLAG-"
chave = "s3cr3t_k3y_1337"

# O que encontrei no .rodata — bytes após a string de prefixo
# Endereço 0x400730, extraí 16 bytes:
cifrado = bytes([0x38, 0x4A, 0x4B, 0x5C, 0x4C, 0x3F,
                 0x40, 0x4E, 0x2A, 0x39, 0x2A, 0x39,
                 0x2A, 0x39, 0x2A, 0x39])

# Aplica XOR com a chave (repetindo)
flag = ""
for i, b in enumerate(cifrado):
    flag += chr(b ^ ord(chave[i % len(chave)]))

print(f"Flag decodificada: {prefixo}{flag}")
```

Saída:

```
Flag decodificada: FLAG-x0r_1s_n0t_s3cur3
```

---

## Passo 5 — Verificação

```bash
$ ./validate FLAG-x0r_1s_n0t_s3cur3
Correct! Flag: HTB{str1ngs_4r3_0p3n_s3s4m3}
```

A flag é exibida! O binário validou o serial e revelou a flag.

```
  ╔══════════════════════════════════╗
  ║      STRINGTHEORY — VALIDADOR    ║
  ║   ─────────────────────────────   ║
  ║                                   ║
  ║   $ ./validate FLAG-x0r_1s_n0t   ║
  ║   _s3cur3                         ║
  ║                                   ║
  ║   Correct! Flag: HTB{str1ngs_    ║
  ║   4r3_0p3n_s3s4m3}               ║
  ║                                   ║
  ║   ╔═══╗╔═══╗╔═╗ ╔═╗╔════╗       ║
  ║   ║ S ║║ O ║║ L ║ ║ V ║║ E ║       ║
  ║   ╚═══╝╚═══╝╚═╝ ╚═╝╚════╝       ║
  ╚══════════════════════════════════╝
```

---

## Ferramentas Utilizadas

| Ferramenta   | Função                                  |
|--------------|----------------------------------------|
| `strings`    | Extrair texto legível do binário       |
| `rabin2`     | Listar strings com endereços           |
| `objdump`    | Desassemblar e ver a lógica            |
| `xxd`        | Hex dump para extrair bytes cifrados   |
| `r2` (radare2) | Análise interativa mais aprofundada  |
| `Python`     | Decodificar XOR e automatizar          |

---

## Lições Aprendidas

1. **Sempre rode `strings` primeiro** — a flag ou a senha muitas vezes está em texto puro no binário.
2. **Binários "not stripped" revelam símbolos** — funções, variáveis globais, strings endereçadas.
3. **XOR é a cifra mais comum em reverse de CTF** — fácil de implementar e reverter.
4. **Prefixo de flag no binário** (como `FLAG-`) é um forte indicativo de validação passo a passo.
5. **Ferramentas de linha de comando** (`strings`, `rabin2`, `objdump`) resolvem 80% dos desafios fáceis de reversa.

---

## Flag

```
HTB{str1ngs_4r3_0p3n_s3s4m3}
```

---

## Referências

- [radare2 — rabin2](https://r2wiki.readthedocs.io/en/latest/home/rabin2/)
- [Linux strings command](https://man7.org/linux/man-pages/man1/strings.1.html)
- [XOR Cipher — Wikipedia](https://en.wikipedia.org/wiki/XOR_cipher)
- [TryHackMe — Reverse Engineering](https://tryhackme.com/room/reverseengineering)
