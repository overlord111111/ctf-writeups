# 🏴 CTF Writeups

Writeups detalhados de Capture The Flag — exploração, engenharia reversa, esteganografia, crypto e desafios web resolvidos passo a passo.

## Categorias

| Categoria | Descrição |
|-----------|-----------|
| 🌐 Web | SQLi, XSS, LFI, SSRF, bypass |
| 🔐 Crypto | Cifras, hashes, força bruta |
| 📁 Estego | Arquivos ocultos, metadados, LSB |
| 🔄 Reversa | Descompilação, análise binária |
| 💻 Forense | Memória, logs, tráfego |

---

## Writeups

### 🌐 [Web — SQL Injection Básico](web/sqli-basic.md)
**Dificuldade:** ★☆☆☆☆ (Fácil)

Bypass de autenticação usando SQL Injection clássico em formulário de login. Inclui enumeração de tecnologia (Apache + PHP), detecção de erro SQL, injeção com `' OR '1'='1' -- -` e captura da flag. Código vulnerável reconstruído e lições sobre prepared statements.

### 🔐 [Crypto — Cifra de César](crypto/caesar-walk.md)
**Dificuldade:** ★☆☆☆☆ (Fácil)

Quebra de cifra de deslocamento (César) com análise manual, brute-force em Python e descoberta de flag em metadados do arquivo original. Aborda ROT13, shift 3, comandos `tr` no terminal.

### 🔄 [Reverse — Engenharia Reversa com Strings](reverse/strings-rev.md)
**Dificuldade:** ★☆☆☆☆ (Fácil)

Análise de binário ELF `validate` usando `strings`, `rabin2` e `objdump`. Descoberta de chave XOR embutida, extração de bytes cifrados e decodificação do serial para obter a flag.

### 🖼️ [Estego — Pixels Ocultos](stego/hidden-pixels.md)
**Dificuldade:** ★★☆☆☆ (Médio)

Esteganografia LSB em imagem PNG. Uso de `zsteg`, `stegsolve` e extração manual em Python para revelar mensagem oculta nos bits menos significativos dos canais RGB. Aborda também iscas em planos de bits e prevenção.

---

## Plataformas

- Hack The Box
- TryHackMe
- CTFtime
- Desafios personalizados

---

## Estrutura do Repositório

```
ctf-writeups/
├── README.md
├── web/
│   └── sqli-basic.md
├── crypto/
│   └── caesar-walk.md
├── reverse/
│   └── strings-rev.md
└── stego/
    └── hidden-pixels.md
```

---

⚡ *"Submit a writeup, learn twice."*
