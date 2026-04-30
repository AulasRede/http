# Roteiro Prático — Fluxo B (Aluno SEM privilégio de administrador)

> **Pré-requisito deste roteiro:** você **não** possui privilégio de administrador na máquina do laboratório. Toda a análise prática será feita sobre tráfego **HTTP em texto claro** — a inspeção de HTTPS é substituída por análise teórica baseada na fundamentação do [`readme.md`](../readme.md).
>
> **Escopo:** captura e inspeção de sites que aceitam HTTP puro (sem redirecionamento forçado para HTTPS). Sem instalação de certificados. Sem alteração de configurações protegidas pelo UAC.
>
> **Fundamentação teórica:** a teoria (estrutura de mensagens, métodos, status codes, cabeçalhos, papel do proxy e decriptação TLS) está no arquivo [`readme.md`](../readme.md) na raiz do repositório. Este roteiro é auto-contido nos aspectos práticos.

---

## Sumário

- [1. Validação do perfil](#1-validação-do-perfil)
- [2. Preparação do Ambiente](#2-preparação-do-ambiente)
  - [2.1. Instalação sem privilégio administrativo](#21-instalação-sem-privilégio-administrativo)
  - [2.2. Desabilitar promoção forçada para HTTPS no navegador](#22-desabilitar-promoção-forçada-para-https-no-navegador)
  - [2.3. Validação do ambiente](#23-validação-do-ambiente)
  - [2.4. Sites de teste que aceitam HTTP puro](#24-sites-de-teste-que-aceitam-http-puro)
- [3. Atividades Práticas](#3-atividades-práticas)
- [4. Questões de Verificação](#4-questões-de-verificação)
- [5. Entrega](#5-entrega)
- [6. Encerramento](#6-encerramento)

---

## 1. Validação do perfil

Confirme que este é o fluxo correto. Abra um prompt do PowerShell e execute:

```powershell
(New-Object System.Security.Principal.WindowsPrincipal([System.Security.Principal.WindowsIdentity]::GetCurrent())).IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator)
```

Se retornou `False` **e** o UAC pede credenciais que você não possui, siga este roteiro. Caso você consiga executar ferramentas "Como administrador", utilize o roteiro do Fluxo A (na pasta irmã `fluxo-a-administrador/`).

---

## 2. Preparação do Ambiente

### 2.1. Instalação sem privilégio administrativo

Você tem pelo menos uma das opções abaixo. Escolha a primeira que funcionar.

**Opção 1 — Fiddler Classic em modo per-user (preferencial se disponível).**

**Download:** https://www.telerik.com/fiddler/fiddler-classic

1. Baixar `FiddlerSetup.exe`.
2. Executar **sem "como administrador"**. Quando o instalador oferecer, escolher a instalação **"Just me"** ou na pasta do usuário (`%LOCALAPPDATA%\Programs\Fiddler`). Algumas versões do instalador respeitam essa escolha sem exigir UAC.
3. Abrir o Fiddler Classic. Confirmar que a captura está ativa (canto inferior esquerdo: **"Capturing"**; atalho **F12** para ligar/desligar).

> Se o instalador ainda assim exigir UAC e você não tiver credenciais de administrador, passar para a Opção 2.

**Opção 2 — mitmproxy portátil (recomendado para Linux/macOS e como fallback no Windows).**

**Download:** https://mitmproxy.org/ (binário portátil, descompactar em pasta do usuário).

1. Extrair em `%USERPROFILE%\mitmproxy\` (Windows) ou `~/mitmproxy/` (Linux/macOS).
2. Executar `mitmweb` (interface web em `http://127.0.0.1:8081`).
3. Configurar o proxy do navegador para `127.0.0.1:8080` (ver 2.2).

**Opção 3 — HTTP Toolkit portátil.**

**Download:** https://httptoolkit.tech/ (opção *AppImage*/portátil).

1. Baixar o binário portátil, executar sem instalar.
2. Usar a opção **"Intercept a fresh Chrome"** (abre um Chrome já pré-configurado sem exigir admin).

**Opção 4 — máquina virtual ou WSL pessoal.**

Se nenhuma das opções anteriores funcionar, execute uma das ferramentas acima em uma VM pessoal ou dentro do WSL, onde você é administrador do sistema convidado. Contate o professor se nenhuma alternativa for viável.

> **Importante.** Em **nenhum caso** instale o certificado raiz do proxy no armazenamento da máquina do laboratório. Este fluxo **não** inspeciona HTTPS.

**Interface do Fiddler Classic** (se optar por ele):

- **Painel da esquerda:** lista de sessões (cada linha = um par request/response).
- **Painel da direita:** abas `Inspectors`, `AutoResponder`, `Composer`, `Filters`.
- Dentro de **Inspectors**, abas horizontais separam **Request** (cima) e **Response** (baixo) com visualizações `Raw`, `Headers`, `WebForms`, `JSON`, `Cookies`, `TextView`.

No Fiddler, acesse **Tools → Options → HTTPS** e **confirme que *Decrypt HTTPS traffic* está DESMARCADO**. Deixe apenas a captura de HTTP ativa.

### 2.2. Desabilitar promoção forçada para HTTPS no navegador

Navegadores modernos promovem silenciosamente `http://` para `https://` via HSTS e "HTTPS-First Mode". Para inspecionar HTTP puro, desabilite esse comportamento **só para a sessão do laboratório**:

- **Chrome/Edge:** abra `chrome://settings/security` e desative temporariamente a opção de sempre usar conexões seguras, se disponível. Em algumas versões, também pode ser necessário abrir `chrome://flags` (ou `edge://flags`), procurar **"HTTPS-First Mode"** ou **"HTTPS-Upgrades"**, definir como **Disabled** e reiniciar o navegador. Os nomes podem variar conforme a versão e políticas do laboratório.
- **Firefox:** `about:preferences#privacy` → seção **Segurança HTTPS-Only** → escolher **"Não ativar o modo HTTPS-Only"**.

> 💡 Mesmo com essas configurações desabilitadas, sites com HSTS **pré-carregado no navegador** (p. ex. `google.com`, `facebook.com`, grandes bancos) continuarão redirecionando para HTTPS. Use apenas os sites listados na seção 2.4, que **não** estão na lista HSTS preload.

### 2.3. Validação do ambiente

Acesse `http://neverssl.com` no navegador. Na lista de sessões do Fiddler, a sessão deve aparecer com:

- **Método `GET`** (e não `CONNECT`).
- Aba **Response → Raw** mostrando HTML legível em texto claro.

Se você vir um `CONNECT` ou a URL na barra de endereço virar `https://`, revise a seção 2.2 e tente outro site da lista abaixo.

### 2.4. Sites de teste que aceitam HTTP puro

Todos foram verificados para aceitar conexões não cifradas:

| # | URL                                            | Uso no roteiro                                                   |
|---|------------------------------------------------|------------------------------------------------------------------|
| 1 | `http://neverssl.com`                          | Página HTML simples — primeira captura                           |
| 2 | `http://example.com`                           | Minimalista, ótimo para análise linha a linha                    |
| 3 | `http://info.cern.ch`                          | Primeiro site da web (1991); curiosidade histórica               |
| 4 | `http://httpbin.org/gzip`                      | Resposta compactada em HTTP puro                                 |
| 5 | `http://httpbin.org/get?...`                   | Eco de request em JSON (aceita HTTP)                             |
| 6 | `http://httpbin.org/forms/post` e `/post`      | Formulários e POST                                               |
| 7 | `http://httpbin.org/cookies/set?...`           | Cookies e sessão                                                 |
| 8 | `http://httpbin.org/user-agent`                | Inspeção de `User-Agent`                                         |
| 9 | `http://httpbin.org/status/{codigo}` e `http://httpbin.org/redirect-to?...` | Geração controlada de códigos de status (200, 301, 404, 500...)  |
| 10| `http://eu.httpbin.org/...`                    | Espelho europeu do httpbin, útil se o principal estiver lento    |
| 11| `http://www.textfiles.com`                     | Site histórico, apenas texto/HTML                                |
| 12| `http://www.columbia.edu/~fdc/sample.html`     | Página acadêmica estável em HTTP                                 |

---

## 3. Atividades Práticas

> **Como registrar.** Para cada atividade, preencha a seção correspondente no arquivo [`relatorio.md`](relatorio.md) desta mesma pasta com:
> - Captura de tela da sessão no Fiddler.
> - Trecho da mensagem *raw* (request ou response) conforme solicitado.
> - Respostas às questões embutidas.

### Atividade 1 — Primeira captura

**Objetivo:** validar o ambiente e familiarizar-se com a interface.

1. Com o Fiddler capturando, abrir o navegador e acessar `http://example.com`.
2. Identificar a sessão correspondente na lista do Fiddler (coluna `Host` = `example.com`).
3. Selecionar a sessão e, no painel `Inspectors`, abrir a aba **Raw** (tanto em Request quanto em Response).

**Registrar no relatório:**
- Captura de tela da sessão selecionada.
- A linha inicial (*request-line*) do pedido enviado.
- A linha inicial (*status-line*) da resposta recebida.

**Pergunta 1.1:** Quantos cabeçalhos o navegador enviou no request? Liste-os.

**Pergunta 1.2:** Qual foi o `Content-Length` da resposta? Se esse cabeçalho não apareceu, registre `Transfer-Encoding`, a versão do protocolo ou outro indício observado. O corpo retornado é HTML, texto puro, JSON ou binário? Como você descobriu?

---

### Atividade 2 — Anatomia de um GET

**Objetivo:** dissecar uma requisição GET com *query string* e correlacioná-la à resposta.

1. Acessar no navegador: `http://httpbin.org/get?aluno=SEU_NOME&curso=redes`.
2. Localizar a sessão no Fiddler.
3. Inspecionar em **Request → Raw** e **Response → JSON**.

**Registrar no relatório:**
- *Request-line* completa, destacando método, *request-target* e versão.
- Valor exato dos cabeçalhos `Host`, `User-Agent` e `Accept`.
- Conteúdo dos campos `args`, `headers` e `origin` no JSON de resposta.

**Pergunta 2.1:** O valor do campo `origin` no JSON retornado corresponde a qual elemento da rede? Por que não é o IP local da sua máquina (em muitos casos)?

**Pergunta 2.2:** Compare o cabeçalho `User-Agent` enviado pelo navegador com o que aparece no JSON da resposta. Eles coincidem? Justifique.

**Pergunta 2.3:** Modifique a URL para `http://httpbin.org/headers`. Liste até três cabeçalhos que o servidor vê mas que **não aparecem explicitamente** na aba Raw do request, e explique de onde eles vêm (dica: o Fiddler pode inserir alguns, o servidor de proxy reverso do httpbin pode inserir outros). Se não encontrar três, registre os que encontrar e explique por que o resultado pode variar.

---

### Atividade 3 — POST e envio de formulário

**Objetivo:** observar o envio de dados no corpo da mensagem e o tipo MIME associado.

1. Acessar `http://httpbin.org/forms/post`.
2. Preencher os campos do formulário (pizza, tamanho, toppings, etc.).
3. Clicar em **Submit order**.
4. Localizar a nova sessão no Fiddler (método **POST** para `/post`).

**Registrar no relatório:**
- *Request-line* do POST.
- Valor de `Content-Type` e `Content-Length` no request.
- Corpo completo do request (aba **Request → Raw**, abaixo da linha em branco).
- Trecho do JSON de resposta mostrando o campo `form`.

**Pergunta 3.1:** Qual o formato do corpo enviado? (`application/x-www-form-urlencoded`, `multipart/form-data` ou outro?) Como esse formato codifica caracteres especiais como espaço e acentos?

**Pergunta 3.2:** Na aba **Request → WebForms** do Fiddler, o conteúdo aparece tabulado. Compare com a aba **Raw**: qual das duas visões corresponde literalmente aos bytes enviados no socket TCP?

**Pergunta 3.3:** Usando a aba **Composer** do Fiddler, envie manualmente um `POST` para `http://httpbin.org/post` com `Content-Type: application/json` e o corpo `{"protocolo":"HTTP","versao":"1.1"}`. Registre a *response* completa. Que campo do JSON de resposta confirma que o servidor recebeu e interpretou corretamente o JSON?

---

### Atividade 4 — Catálogo de status codes

**Objetivo:** provocar e identificar códigos de status representativos.

Use o serviço `httpbin.org`, que possui endpoints próprios para provocar códigos de status controlados. Para o `301`, use o endpoint de redirecionamento indicado, pois ele retorna também o cabeçalho `Location`.

Para cada URL abaixo, acesse no navegador e localize a sessão no Fiddler:

> No caso do `301`, o navegador provavelmente seguirá o redirecionamento e criará uma segunda sessão para o destino (`/get`). Para esta atividade, registre a sessão original que retornou `301 Moved Permanently`.

| # | URL                                   | Classe esperada |
|---|---------------------------------------|-----------------|
| 1 | `http://httpbin.org/status/200`       | 2xx             |
| 2 | `http://httpbin.org/redirect-to?status_code=301&url=/get` | 3xx |
| 3 | `http://httpbin.org/status/404`       | 4xx             |
| 4 | `http://httpbin.org/status/418`       | 4xx (humorado)  |
| 5 | `http://httpbin.org/status/500`       | 5xx             |
| 6 | `http://httpbin.org/status/503`       | 5xx             |

Adicionalmente, para observar um **304 Not Modified** de forma controlada:
7. Acesse `http://example.com`, aguarde carregar e anote o cabeçalho `Last-Modified` da resposta.
8. No **Composer** do Fiddler (ou ferramenta equivalente), envie um `GET` para `http://example.com/` incluindo o cabeçalho `If-Modified-Since` com o valor exato de `Last-Modified` observado no passo anterior.
9. Localize a sessão condicional gerada no passo 8 e confirme que a resposta retornou `304 Not Modified`. Use esta sessão condicional como a linha 7 da tabela. Se o servidor retornar `200 OK`, registre os cabeçalhos de cache observados e explique a diferença.

**Registrar no relatório:**
- Tabela com as 7 sessões: método, URL, *status-line* completa, `Content-Length` ou `Transfer-Encoding` quando presentes, presença ou ausência de body.

**Pergunta 4.1:** Em qual dos status acima o corpo da resposta está **ausente** ou tem tamanho zero? Isso é obrigatório pela especificação ou depende do servidor?

**Pergunta 4.2:** No caso do `301`, qual cabeçalho da resposta informa ao navegador para onde ir? O que aconteceria se esse cabeçalho não estivesse presente?

**Pergunta 4.3:** Explique a diferença semântica entre `200`, `304` e `404` do ponto de vista do cache do navegador.

---

### Atividade 5 — Identificação de cabeçalhos

**Objetivo:** reconhecer o propósito funcional de cada cabeçalho em tráfego real.

1. Acessar `http://httpbin.org/response-headers?Cache-Control=max-age%3D3600&Set-Cookie=teste%3D1` para provocar cabeçalhos variados em uma resposta controlada.
2. Acessar `http://httpbin.org/gzip` para observar uma resposta compactada em HTTP puro.
3. Recarregar a primeira URL uma vez, para verificar se o cookie `teste=1` passa a aparecer no cabeçalho `Cookie` do request.
4. Selecionar cada sessão e, na aba **Inspectors → Headers**, analisar request e response.

**Preencher a tabela abaixo no relatório** com os valores reais observados, agregando dados das sessões acima conforme o cabeçalho apareça:

| Cabeçalho            | Presente em (Req/Resp)? | Valor capturado | Função em uma frase |
|----------------------|-------------------------|-----------------|---------------------|
| `Host`               |                         |                 |                     |
| `User-Agent`         |                         |                 |                     |
| `Accept`             |                         |                 |                     |
| `Accept-Encoding`    |                         |                 |                     |
| `Cookie`             |                         |                 |                     |
| `Server`             |                         |                 |                     |
| `Content-Type`       |                         |                 |                     |
| `Content-Encoding`   |                         |                 |                     |
| `Set-Cookie`         |                         |                 |                     |
| `Cache-Control`      |                         |                 |                     |
| `Strict-Transport-Security` |                  |                 |                     |

**Pergunta 5.1:** O servidor retornou `Content-Encoding: gzip` (ou `br`)? Se sim, compare o valor de `Content-Length`, quando presente, com o tamanho do conteúdo visível na aba **Response → TextView**. O que explica a diferença?

**Pergunta 5.2:** O que acontece com um request onde o cliente envia `Accept: application/json` mas o recurso só existe em `text/html`? (Pesquisar o código de status correto.)

**Pergunta 5.3:** O cabeçalho `Strict-Transport-Security` apareceu nas respostas HTTP observadas? **Por que esse cabeçalho está ausente neste fluxo?** (Dica: consulte a RFC 6797 sobre em qual protocolo o HSTS é válido.) Explique o papel do HSTS contra downgrades para HTTP puro, ainda que você não o tenha observado em tráfego real.

---

### Atividade 6 — HTTP vs HTTPS (análise comparativa sem decriptação)

**Objetivo:** observar a diferença entre tráfego claro e cifrado, mesmo sem poder decifrar o HTTPS.

1. Acessar `http://neverssl.com` (HTTP puro).
2. Acessar `https://httpbin.org/get` no mesmo navegador.
3. Comparar as duas sessões no Fiddler.

**Registrar no relatório:**
- Captura de tela de cada sessão com a aba **Raw** aberta.
- Para o HTTP, transcrever parte do request e da response (legíveis).
- Para o HTTPS, descrever **o que aparece** na sessão (método, host, bytes cifrados em `TextView`, aba `Inspectors → Statistics` mostrando tamanho e duração).

**Pergunta 6.1:** No caso do `https://httpbin.org/get`, que método HTTP aparece na sessão do Fiddler? Explique o que esse método faz e por que ele existe. (Dica: `CONNECT` é definido na RFC 9110 §9.3.6.)

**Pergunta 6.2:** Faça uma **tabela comparativa** dos campos visíveis ao Fiddler em cada caso:

| Campo                          | Visível em HTTP? | Visível em HTTPS (sem decriptação)? |
|--------------------------------|------------------|-------------------------------------|
| Método                         |                  |                                     |
| URL completa (path + query)    |                  |                                     |
| Cabeçalhos de request          |                  |                                     |
| Corpo de request               |                  |                                     |
| Status code                    |                  |                                     |
| Cabeçalhos de response         |                  |                                     |
| Corpo de response              |                  |                                     |
| Host (via SNI, no `CONNECT`)   |                  |                                     |
| IP e porta de destino          |                  |                                     |

**Pergunta 6.3 (teórica):** Descreva em um parágrafo, com base na fundamentação teórica do [`readme.md`](../readme.md) (seção 4.6), o que você **veria** no Fiddler se tivesse privilégio de administrador e pudesse habilitar *Decrypt HTTPS traffic*. Indique as telas e abas (Inspectors → Raw, Response → JSON/TextView, etc.) onde cada informação apareceria, e **por que** essa inspeção exige a instalação de um certificado raiz no sistema.

**Pergunta 6.4:** Do ponto de vista de segurança, explique por que a técnica de decriptação usada pelos *debugging proxies* **não** funcionaria contra um usuário se um atacante a tentasse sem instalar o certificado na máquina-alvo. Relacione com o conceito de "cooperação do usuário" da seção 4.6 do [`readme.md`](../readme.md).

---

### Atividade 7 — Cookies e sessão

**Objetivo:** rastrear o ciclo de vida de um cookie ao longo de múltiplas requisições.

1. Em uma janela anônima, acessar `http://httpbin.org/cookies/set?disciplina=redes&professor=claudio`.
2. Observar o **`Set-Cookie`** na resposta. Anotar os valores.
3. Seguir o redirecionamento automático para `http://httpbin.org/cookies` (pode ser necessário um clique).
4. Observar o cabeçalho **`Cookie`** enviado nesta segunda requisição.
5. Recarregar mais duas vezes a página `http://httpbin.org/cookies`.

**Registrar no relatório:**
- Sequência (numerada) das sessões capturadas.
- Valores dos cabeçalhos `Set-Cookie` e `Cookie` em cada uma.

**Pergunta 7.1:** O cabeçalho `Set-Cookie` só aparece uma vez ou em toda requisição? Justifique.

**Pergunta 7.2:** Que atributos o `Set-Cookie` trouxe (`Path`, `Domain`, `Expires`, `Max-Age`, `Secure`, `HttpOnly`, `SameSite`)? Para cada um presente, explique brevemente sua função. Se algum atributo da lista não apareceu, registre como **não observado**.

**Pergunta 7.3:** O atributo `Secure` **pode** aparecer em um cookie recebido por HTTP puro como neste exercício? Qual o comportamento esperado do navegador se um cookie `Secure` **fosse** enviado nesta conexão? Relacione com o fato de que todo o tráfego desta atividade é **visível em texto claro** ao Fiddler (e, portanto, a qualquer observador na rede).

**Pergunta 7.4:** Na aba **Inspectors → Cookies** do Fiddler, compare o cookie armazenado com o que o servidor devolve no campo `cookies` do JSON da resposta. Eles coincidem?

---

### Atividade 8 — Manipulação com *breakpoints*

**Objetivo:** compreender o proxy como agente ativo, capaz de modificar tráfego em trânsito.

1. No Fiddler, habilitar **breakpoint de request**: menu **Rules → Automatic Breakpoints → Before Requests** (atalho **F11**).
2. No navegador, acessar `http://httpbin.org/user-agent`.
3. A sessão pausará no Fiddler com um ícone vermelho.
4. Na aba **Inspectors → Request → Raw**, editar o cabeçalho `User-Agent` para um valor inventado, por exemplo:
   `User-Agent: LaboratorioRedes/1.0 (Aluno NOME)`
5. Clicar em **Run to Completion** (botão verde no topo do inspector).
6. Na resposta, observar o JSON retornado.

**Registrar no relatório:**
- Captura de tela da edição do request.
- Trecho do JSON de resposta mostrando o campo `user-agent`.

**Pergunta 8.1:** O servidor tem como detectar que o `User-Agent` foi forjado? Discuta.

**Pergunta 8.2:** Repita o exercício, mas desta vez habilite **breakpoint de response** (**Rules → Automatic Breakpoints → After Responses**). Acesse `http://httpbin.org/status/200` e, quando o Fiddler pausar, edite a *status-line* para `HTTP/1.1 404 Not Found`. Libere. O que o navegador exibe? A manipulação afetou apenas a visualização local — comente sobre o papel do proxy como *man-in-the-middle*.

**Pergunta 8.3:** Desabilite todos os breakpoints (**Rules → Automatic Breakpoints → Disabled**, atalho **Shift+F11**) ao terminar.

---

### Atividade 9 — Redirecionamento HTTP → HTTPS (exclusiva do Fluxo B)

**Objetivo:** observar, no próprio Fiddler, um redirecionamento controlado de HTTP para HTTPS e relacioná-lo aos mecanismos de segurança discutidos na teoria.

1. Com os breakpoints desabilitados, acessar no navegador: `http://httpbin.org/redirect-to?status_code=301&url=https%3A%2F%2Fhttpbin.org%2Fget`.
2. Localizar no Fiddler a sessão original para `httpbin.org` que retornou `301 Moved Permanently`.
3. Observar que o navegador pode criar uma segunda sessão para o destino `https://httpbin.org/get`; para esta atividade, registre a sessão original do redirecionamento.

**Registrar no relatório:**
- Captura de tela da sessão.
- *Status-line* completa da resposta.
- Cabeçalho `Location` da resposta (se presente).

**Pergunta 9.1:** Qual foi o **código de status** retornado e qual **cabeçalho** direcionou o navegador para `https://`?

**Pergunta 9.2:** Com base na fundamentação teórica do [`readme.md`](../readme.md) (seções 4.5 e Anexo A), além do redirecionamento 3xx, **qual outro mecanismo** faz o navegador passar a forçar HTTPS em visitas futuras, mesmo que o usuário digite `http://`? Responda citando o cabeçalho de response envolvido e a RFC que o define.

**Pergunta 9.3:** Se o servidor enviasse o cabeçalho citado em 9.2 por uma resposta servida via **HTTP puro** (porta 80), o navegador deveria obedecer? Justifique com base na RFC.

---

## 4. Questões de Verificação

Responda no relatório. Respostas concisas, com base no observado, são preferíveis a explicações genéricas.

1. Qual é a **ordem dos elementos** em uma mensagem HTTP/1.1? O que separa os cabeçalhos do corpo?
2. Por que o cabeçalho `Host` é **obrigatório** em HTTP/1.1 mas era opcional em HTTP/1.0? (Dica: virtual hosting.)
3. Explique a diferença entre os códigos `401 Unauthorized` e `403 Forbidden`.
4. Um `POST` enviado duas vezes produz o mesmo efeito que um único envio? E um `PUT`? Justifique em termos de **idempotência**.
5. Por que HTTPS, mesmo cifrando todo o tráfego de aplicação, ainda permite que um observador saiba **qual site** o usuário está visitando? (Cite SNI e DNS.)
6. O que `Content-Encoding: gzip` muda no fluxo? Em que ponto os dados são compactados e em que ponto são descompactados?
7. Um servidor envia `Cache-Control: no-store` em uma resposta. Qual o impacto prático no comportamento do navegador?
8. Com base na fundamentação teórica (seção 4.6 do [`readme.md`](../readme.md)), descreva em até 5 linhas como um *debugging proxy* consegue decifrar HTTPS sem violar a criptografia — e por que isso **só é possível com cooperação do usuário** (e por que, justamente por isso, você não pôde executar essa etapa).
9. Dê um exemplo concreto, observado nas atividades, de um cabeçalho de **request** que o navegador envia automaticamente, sem a página pedir.
10. Se você quisesse automatizar a inspeção (script), qual das ferramentas alternativas da seção 2.1 seria mais adequada? Por quê?
11. (**Exclusiva do Fluxo B**) Liste **três cabeçalhos de segurança** que **não aparecem ou não fazem sentido** em respostas servidas por HTTP puro (p. ex. `Strict-Transport-Security`, `Content-Security-Policy` com diretivas `upgrade-insecure-requests`, `Set-Cookie; Secure`). Para cada um, explique o que aconteceria se fosse enviado por um servidor HTTP: alguns seriam **ignorados** pelo navegador, outros só têm efeito quando recebidos **via** HTTPS. Cite a RFC 6797 para o caso do HSTS.

---

## 5. Entrega

### O que entregar
- Arquivo **`relatorio.md`** preenchido (template nesta mesma pasta).
- Pasta **`evidencias/`** com as capturas de tela nomeadas por atividade (`atv1_sessao.png`, `atv3_post_raw.png`, etc.), incluindo a captura da Atividade 9 (redirecionamento controlado de `http://httpbin.org/...` para `https://httpbin.org/get`).

### Como entregar
- Compactar a pasta do aluno em `NOME_RA_LAB_HTTP_FLUXOB.zip`.
- Submeter no **Microsoft Teams**, atividade correspondente, até a data definida em aula.

---

## 6. Encerramento


1. **Reabilitar** o HTTPS-First Mode / HTTPS-Only Mode no navegador (passos inversos de 2.2), para que seu dia-a-dia volte a priorizar conexões seguras.
2. **Fechar o Fiddler** (ou `mitmproxy` / HTTP Toolkit) para liberar a porta de proxy e remover qualquer configuração de proxy do navegador.
3. **Anexar ao relatório** um parágrafo explicando **por que**, neste fluxo, a etapa de remoção de certificado é **dispensável** — e **por que**, no fluxo do aluno administrador, ela seria **obrigatória**. Use a seção 4.6 do [`readme.md`](../readme.md) como referência.
