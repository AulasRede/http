# Laboratório: Inspeção de HTTP/HTTPS com *Debugging Proxy*

> **Disciplina:** Redes de Computadores
> **Professor:** Claudio Nunes
> **Tópico:** Camada de Aplicação — Protocolos HTTP/1.1 e HTTP/2 sobre TLS

Este documento contém a **fundamentação teórica** da atividade e direciona cada aluno para o roteiro prático apropriado, de acordo com seu nível de privilégio na máquina de laboratório.

---

## Sumário

- [1. Objetivos de Aprendizagem](#1-objetivos-de-aprendizagem)
- [2. Pré-requisitos](#2-pré-requisitos)
- [3. Escolha do Roteiro Prático](#3-escolha-do-roteiro-prático)
- [4. Fundamentação Teórica](#4-fundamentação-teórica)
  - [4.1. O protocolo HTTP](#41-o-protocolo-http)
  - [4.2. Anatomia de uma mensagem HTTP](#42-anatomia-de-uma-mensagem-http)
  - [4.3. Métodos (verbos) HTTP](#43-métodos-verbos-http)
  - [4.4. Códigos de status](#44-códigos-de-status)
  - [4.5. Principais cabeçalhos](#45-principais-cabeçalhos)
  - [4.6. O papel do *debugging proxy*](#46-o-papel-do-debugging-proxy)
- [5. Referências](#5-referências)
- [Anexo A — Tabela-resumo de cabeçalhos](#anexo-a--tabela-resumo-de-cabeçalhos)
- [Anexo B — Tabela-resumo de status codes](#anexo-b--tabela-resumo-de-status-codes)

---

## 1. Objetivos de Aprendizagem

Ao final desta atividade, o aluno será capaz de:

1. **Descrever** a estrutura de uma mensagem HTTP (linha inicial, cabeçalhos, corpo) em nível de bytes.
2. **Identificar** os métodos, os códigos de status e os principais cabeçalhos de requests/responses reais.
3. **Utilizar** um *debugging proxy* (Fiddler) para capturar, inspecionar e manipular tráfego HTTP.
4. **Diferenciar** o comportamento de HTTP e HTTPS na camada de transporte.
5. **Analisar** fluxos completos como navegação web, submissão de formulário e manutenção de sessão via cookies.

---

## 2. Pré-requisitos

### Conhecimentos prévios
- Modelo TCP/IP e camada de aplicação.
- Conceito de cliente/servidor e sockets TCP.
- Noções básicas de TLS/SSL.

### Ambiente técnico
- Sistema operacional: Windows 10/11 (recomendado para Fiddler Classic) ou Linux/macOS (usar mitmproxy/HTTP Toolkit).
- Navegador Chrome, Firefox ou Edge atualizado.
- Acesso à internet sem bloqueios a sites de teste (`httpbin.org`, `example.com`, `neverssl.com`).

### Material
- Template `relatorio.md` presente na pasta do roteiro escolhido.
- Ferramenta de captura de tela para anexar prints ao relatório.

---

## 3. Escolha do Roteiro Prático

A inspeção de tráfego **HTTPS** pelo Fiddler exige a instalação de um **certificado raiz** no armazenamento de confiança do sistema operacional. Essa operação **requer privilégios de administrador**. Para acomodar laboratórios mistos, há dois roteiros independentes e auto-contidos:

| Fluxo | Quem usa | Escopo                                     | Pasta |
|-------|----------|--------------------------------------------|-------|
| **A** | Alunos **COM** privilégio de administrador | Captura e inspeção de HTTP **e HTTPS** (com decriptação TLS) | [`fluxo-a-administrador/`](fluxo-a-administrador/roteiro.md) |
| **B** | Alunos **SEM** privilégio de administrador | Captura e inspeção somente de **HTTP em texto claro** + análise teórica de HTTPS | [`fluxo-b-sem-administrador/`](fluxo-b-sem-administrador/roteiro.md) |

> **Teste rápido para decidir o fluxo.** Abra um prompt do PowerShell e execute:
> ```powershell
> (New-Object System.Security.Principal.WindowsPrincipal([System.Security.Principal.WindowsIdentity]::GetCurrent())).IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator)
> ```
> - Retornou `True` **ou** você consegue executar o Fiddler "Como administrador" sem que o UAC exija senha de outra conta → siga o **[Fluxo A](fluxo-a-administrador/roteiro.md)**.
> - Retornou `False` **e** o UAC pede credenciais que você não possui → siga o **[Fluxo B](fluxo-b-sem-administrador/roteiro.md)**.

> 📝 **Nota sobre avaliação.** Os dois fluxos têm **peso equivalente** na nota final. O Fluxo B inclui questões teóricas adicionais que compensam a ausência da inspeção prática de HTTPS. Cada aluno deve escolher **um único fluxo** no início da aula e segui-lo até o final — não há necessidade de consultar o outro roteiro.

---

## 4. Fundamentação Teórica

### 4.1. O protocolo HTTP

O **HyperText Transfer Protocol** (HTTP) é um protocolo de camada de aplicação, baseado em texto (até HTTP/1.1), sem estado, do tipo request/response. Opera por padrão sobre TCP:

| Versão    | Transporte            | Porta padrão | Característica-chave                                    |
|-----------|-----------------------|--------------|---------------------------------------------------------|
| HTTP/1.0  | TCP                   | 80           | Uma conexão por request                                 |
| HTTP/1.1  | TCP                   | 80 (HTTPS 443) | *Keep-alive*, *pipelining*, chunked encoding          |
| HTTP/2    | TCP + TLS (ALPN `h2`) | 443          | Binário, multiplexação, *header compression* (HPACK)    |
| HTTP/3    | QUIC (UDP + TLS 1.3)  | 443          | Sem *head-of-line blocking*, estabelecimento mais rápido |

**HTTPS** é HTTP encapsulado em TLS. Toda a mensagem HTTP — linha de status, headers e body — é cifrada. Em um cenário típico, sem tecnologias adicionais como ECH (*Encrypted Client Hello*) ou DNS cifrado, IP, porta, DNS e SNI (nome do host) ainda podem ficar visíveis a um observador externo.

### 4.2. Anatomia de uma mensagem HTTP

Toda mensagem HTTP/1.x tem a mesma estrutura:

```
<linha inicial>            ← request-line OU status-line
<Header-1>: <valor>        ← zero ou mais linhas de cabeçalho
<Header-2>: <valor>
...
<linha em branco>          ← CRLF separador
<corpo opcional>           ← body (pode estar ausente)
```

**Exemplo de request** capturado num navegador:

```http
GET /get?id=42 HTTP/1.1
Host: httpbin.org
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Encoding: gzip, deflate, br
Accept-Language: pt-BR,pt;q=0.9,en;q=0.8
Connection: keep-alive
```

**Exemplo de response** correspondente:

```http
HTTP/1.1 200 OK
Date: Wed, 22 Apr 2026 18:23:14 GMT
Content-Type: application/json
Content-Length: 287
Server: gunicorn/19.9.0
Access-Control-Allow-Origin: *
Connection: keep-alive

{
  "args": {"id": "42"},
  "headers": { ... },
  "origin": "200.100.50.30",
  "url": "https://httpbin.org/get?id=42"
}
```

### 4.3. Métodos (verbos) HTTP

| Método   | Função                                          | Possui body? | Idempotente? | Seguro? |
|----------|-------------------------------------------------|--------------|--------------|---------|
| `GET`    | Recuperar representação de um recurso           | Não          | Sim          | Sim     |
| `HEAD`   | Como GET, mas só retorna cabeçalhos             | Não          | Sim          | Sim     |
| `POST`   | Submeter dados (formulários, APIs)              | Sim          | Não          | Não     |
| `PUT`    | Criar/substituir recurso em URI conhecida       | Sim          | Sim          | Não     |
| `PATCH`  | Atualização parcial                             | Sim          | Não          | Não     |
| `DELETE` | Remover recurso                                 | Opcional     | Sim          | Não     |
| `OPTIONS`| Descobrir métodos/CORS suportados               | Não          | Sim          | Sim     |

> **Seguro** (*safe*): não altera estado no servidor.
> **Idempotente**: múltiplas execuções produzem o mesmo efeito que uma única.

### 4.4. Códigos de status

Agrupados por classe (primeiro dígito):

| Classe | Faixa       | Significado            | Exemplos frequentes                       |
|--------|-------------|------------------------|-------------------------------------------|
| 1xx    | 100–199     | Informacional          | `100 Continue`, `101 Switching Protocols` |
| 2xx    | 200–299     | Sucesso                | `200 OK`, `201 Created`, `204 No Content` |
| 3xx    | 300–399     | Redirecionamento       | `301 Moved Permanently`, `302 Found`, `304 Not Modified` |
| 4xx    | 400–499     | Erro do cliente        | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests` |
| 5xx    | 500–599     | Erro do servidor       | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

### 4.5. Principais cabeçalhos

**De request** (cliente → servidor):

| Cabeçalho          | Propósito                                                            |
|--------------------|----------------------------------------------------------------------|
| `Host`             | Nome de domínio do servidor (obrigatório em HTTP/1.1)                |
| `User-Agent`       | Identificação do cliente/navegador                                   |
| `Accept`           | Tipos MIME aceitos na resposta                                       |
| `Accept-Encoding`  | Algoritmos de compressão suportados (`gzip`, `br`, `deflate`)        |
| `Accept-Language`  | Idiomas preferidos                                                   |
| `Cookie`           | Cookies previamente recebidos                                        |
| `Authorization`    | Credenciais (`Basic`, `Bearer <token>`, etc.)                        |
| `Referer`          | URL de origem da requisição                                          |
| `Content-Type`     | MIME do corpo enviado (em POST/PUT)                                  |
| `Content-Length`   | Tamanho do corpo em bytes quando o tamanho é conhecido antecipadamente |
| `If-None-Match`    | Validação de cache por ETag                                          |

**De response** (servidor → cliente):

| Cabeçalho          | Propósito                                                            |
|--------------------|----------------------------------------------------------------------|
| `Server`           | Identificação do software servidor                                   |
| `Content-Type`     | MIME do corpo retornado                                              |
| `Content-Length`   | Tamanho do corpo em bytes quando o tamanho é conhecido antecipadamente |
| `Content-Encoding` | Compressão aplicada ao corpo                                         |
| `Set-Cookie`       | Cookie a ser armazenado pelo cliente                                 |
| `Cache-Control`    | Política de cache (`no-store`, `max-age=3600`, `public`)             |
| `ETag`             | Identificador de versão do recurso                                   |
| `Location`         | URL para onde redirecionar (usado em 3xx e 201)                      |
| `WWW-Authenticate` | Esquema de autenticação exigido (em 401)                             |
| `Strict-Transport-Security` | Força uso de HTTPS em acessos futuros (HSTS)                |

### 4.6. O papel do *debugging proxy*

Um *debugging proxy* (como Fiddler, mitmproxy, Burp, HTTP Toolkit) é um **man-in-the-middle controlado** que o próprio usuário instala:

```
[Navegador]  ──►  [Proxy local na porta 8888]  ──►  [Internet]  ──►  [Servidor]
                          │
                          ▼
                 [Interface de inspeção]
```

Para tráfego HTTPS, o proxy:
1. Gera um **certificado raiz próprio**, instalado pelo usuário no armazenamento de confiança.
2. Emite certificados "falsos" assinados por essa raiz para cada site visitado.
3. Decifra a comunicação no meio e reencripta antes de enviar.

> ⚠️ **Atenção ética.** O certificado raiz do Fiddler, quando instalado, permite a decriptografia de **qualquer** conexão HTTPS feita pela máquina. Ele deve permanecer **apenas** na máquina do estudante e ser **removido ao final do laboratório** — essa exigência está detalhada no roteiro do Fluxo A.

---

## 5. Referências

- **RFC 9110** — HTTP Semantics. https://datatracker.ietf.org/doc/html/rfc9110
- **RFC 9111** — HTTP Caching. https://datatracker.ietf.org/doc/html/rfc9111
- **RFC 9112** — HTTP/1.1. https://datatracker.ietf.org/doc/html/rfc9112
- **RFC 9113** — HTTP/2. https://datatracker.ietf.org/doc/html/rfc9113
- **RFC 6265** — HTTP State Management Mechanism (Cookies). https://datatracker.ietf.org/doc/html/rfc6265
- **RFC 6797** — HTTP Strict Transport Security (HSTS). https://datatracker.ietf.org/doc/html/rfc6797
- MDN Web Docs — HTTP. https://developer.mozilla.org/en-US/docs/Web/HTTP
- Fiddler Classic — Documentação oficial. https://docs.telerik.com/fiddler/
- httpbin — Serviço de teste HTTP. https://httpbin.org/
- KUROSE, J.; ROSS, K. *Redes de Computadores e a Internet*, 8ª ed., Cap. 2 — Camada de Aplicação.
- TANENBAUM, A. S.; FEAMSTER, N.; WETHERALL, D. *Redes de Computadores*, 6ª ed., Cap. 7 — Camada de Aplicação.

---

## Anexo A — Tabela-resumo de cabeçalhos

| Cabeçalho                    | Direção  | Exemplo de valor                            |
|------------------------------|----------|---------------------------------------------|
| `Host`                       | Req      | `www.example.com`                           |
| `User-Agent`                 | Req      | `Mozilla/5.0 (...)`                         |
| `Accept`                     | Req      | `text/html, */*; q=0.1`                     |
| `Accept-Encoding`            | Req      | `gzip, deflate, br`                         |
| `Accept-Language`            | Req      | `pt-BR, pt; q=0.9, en; q=0.5`               |
| `Cookie`                     | Req      | `sess=abc123; lang=pt`                      |
| `Authorization`              | Req      | `Bearer eyJhbGci...`                        |
| `Referer`                    | Req      | `https://origem.com/pagina`                 |
| `Content-Type`               | Ambos    | `application/json; charset=utf-8`           |
| `Content-Length`             | Ambos    | `1342`                                      |
| `If-None-Match`              | Req      | `"686897696a7c876b7e"`                      |
| `Server`                     | Resp     | `nginx/1.25.3`                              |
| `Set-Cookie`                 | Resp     | `sess=xyz; Path=/; Secure; HttpOnly`        |
| `Cache-Control`              | Resp     | `max-age=3600, public`                      |
| `ETag`                       | Resp     | `"686897696a7c876b7e"`                      |
| `Location`                   | Resp     | `https://destino.com/novo`                  |
| `Strict-Transport-Security`  | Resp     | `max-age=31536000; includeSubDomains`       |
| `Content-Encoding`           | Resp     | `gzip`                                      |
| `WWW-Authenticate`           | Resp     | `Basic realm="Admin"`                       |

---

## Anexo B — Tabela-resumo de status codes

| Código | Frase de razão          | Quando ocorre                                    |
|--------|-------------------------|--------------------------------------------------|
| 100    | Continue                | Resposta intermediária a `Expect: 100-continue`  |
| 200    | OK                      | Requisição bem-sucedida com corpo                |
| 201    | Created                 | Recurso criado (comum em `POST`/`PUT`)           |
| 204    | No Content              | Sucesso sem corpo de resposta                    |
| 301    | Moved Permanently       | Recurso migrou definitivamente                   |
| 302    | Found                   | Redirecionamento temporário                      |
| 304    | Not Modified            | Cache do cliente ainda válido                    |
| 400    | Bad Request             | Sintaxe inválida no request                      |
| 401    | Unauthorized            | Autenticação ausente ou inválida                 |
| 403    | Forbidden               | Autenticado mas sem permissão                    |
| 404    | Not Found               | Recurso inexistente                              |
| 405    | Method Not Allowed      | Método não suportado no recurso                  |
| 408    | Request Timeout         | Cliente demorou a enviar                         |
| 418    | I'm a teapot            | RFC 2324 (piada tradicional)                     |
| 429    | Too Many Requests       | Rate limiting                                    |
| 500    | Internal Server Error   | Erro genérico do servidor                        |
| 502    | Bad Gateway             | Gateway/proxy recebeu resposta inválida          |
| 503    | Service Unavailable     | Sobrecarga ou manutenção                         |
| 504    | Gateway Timeout         | Gateway não obteve resposta do upstream          |

---

**Versão:** 2.0 — Abr/2026
**Licença:** CC BY-NC-SA 4.0
