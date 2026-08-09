# Diário de Bordo — Orientações

Página estática (single-file HTML) para cadastrar, organizar e consultar
**orientações de avaliação** (tema + versão simples + versão com orientação
futura), agrupadas em 4 categorias fixas. Pensada para uso via GitHub Pages,
sem backend próprio.

## Funcionalidades

- Login por endereço da API + chave (`X-API-Key`), salvos só no navegador.
- Listagem das orientações agrupadas por categoria, com busca por tema/texto.
- Criar, editar e excluir orientações.
- Importar/exportar orientações em JSON, com checagem de duplicados
  (mesmo tema + categoria) no frontend.
- Logout (⏻) limpando a configuração salva localmente.

## Como usar

Basta abrir `index.html` no navegador (ou publicar via GitHub Pages).
Na primeira execução, informe o endereço da API e a chave de acesso — esses
dados ficam salvos em `localStorage`, apenas no navegador de quem usa.

## Backend

Esta página **não tem backend próprio**. Ela usa a API do **Prompt Master**
(`https://smlapi.duckdns.org/prompts`), que já roda na VM2
(`instance-projetoept-02`, porta 5008, systemd `prompt-master.service`),
reaproveitando a tabela `prompts` existente em vez de subir um serviço novo.

Cada orientação de avaliação é armazenada como um registro de "prompt"
comum, distinguido dos demais prompts do usuário por uma tag marcadora
(`diario-orientacao`).

### Autenticação

- A página exige **endereço da API + chave (`X-API-Key`)** antes de liberar o uso.
- Endpoint e chave ficam salvos em `localStorage` (`diario-orientacoes-config`),
  só no navegador de quem usa — nunca no código-fonte publicado.
- A chave real vive em `backend/.env` na VM2 (`API_KEY=...`), fora do repositório.
- Botão de logout (⏻) limpa o `localStorage` e volta pra tela de login.

### Modelo de dados (adapter)

A API do Prompt Master usa o schema `PromptIn` (Pydantic/FastAPI):

```python
titulo: str
prompt: str
categoria: str = "Geral"
tags: list[str] = []   # não confundir com as colunas internas tag1/tag2/tag3
```

Mapeamento orientação → prompt:

| Campo da orientação                | Campo na API                                           |
|-------------------------------------|---------------------------------------------------------|
| `tema`                               | `titulo`                                                |
| `categoria_id` (1-4)                 | `categoria` (nome exato de uma das 4 categorias fixas)  |
| `versao_simples` + `versao_futura`   | `prompt`, empacotados no texto (ver abaixo)             |
| — (marcador interno)                 | `tags: ["diario-orientacao"]`                           |

**Empacotamento do campo `prompt`** — as duas versões viram um texto só,
no mesmo formato que já era usado no prompt de revisão original:

```
VERSÃO SIMPLES:
<texto>
VERSÃO COM ORIENTAÇÃO FUTURA:
<texto>
```

Um regex no frontend (`desempacotar()`) separa isso de volta em duas
strings ao carregar da API.

**Filtro por tag** — `GET /api/prompts` retorna *todos* os prompts do
usuário (inclusive os que nada têm a ver com o diário). O frontend filtra
client-side por `tags.includes("diario-orientacao")` antes de exibir
qualquer coisa. Sem essa tag, um item existe na API mas fica invisível na
página.

### CORS

O Prompt Master usa `CORSMiddleware` (FastAPI), configurado via
`CORS_ORIGINS` em `backend/.env` (lista separada por vírgula). Qualquer
domínio novo que for consumir essa API — como o GitHub Pages desta
página — **precisa ser adicionado nessa variável**, senão o preflight
`OPTIONS` falha com `No 'Access-Control-Allow-Origin' header`.

```
CORS_ORIGINS=https://hyskal.github.io,https://projetos-ept.github.io
```

Depois de editar, reiniciar o serviço (`pmrestart`) — o valor só é lido
na inicialização do processo.

### Limitações conhecidas

- **Sem transação/dedupe no backend.** A checagem de duplicado (mesmo
  tema + categoria) acontece só no frontend, comparando com o que já
  está carregado na tela. Duas abas importando o mesmo JSON ao mesmo
  tempo podem gerar duplicatas.
- **Nenhuma soft-delete.** Excluir um item chama `DELETE /api/prompts/{id}`
  direto — não tem lixeira nem desfazer.
- **Categoria por string exata.** Se o nome salvo em `categoria` não bater
  caractere por caractere com um dos 4 nomes fixos, o item cai por padrão
  na categoria 1 ao ser relido.

### Histórico de bugs corrigidos (pra não repetir)

1. **CORS ausente** para o domínio do GitHub Pages novo → corrigido
   adicionando o domínio em `CORS_ORIGINS`.
2. **Campo de tag errado.** Uma versão inicial do adapter mandava
   `tag1: "diario-orientacao"` (nome de coluna interna do banco) em vez de
   `tags: ["diario-orientacao"]` (campo real aceito pelo schema `PromptIn`).
   Isso criava registros "órfãos" — salvos, mas sem tag, logo invisíveis
   no filtro. Corrigido trocando o campo enviado no `POST`/`PATCH`.
   Os órfãos gerados por essa falha (ids 29–73 no banco) tiveram a tag
   corrigida retroativamente via `UPDATE` direto no SQLite.

### Se algum dia precisar migrar para uma API própria

Toda a comunicação passa por 4 funções isoladas no HTML
(`apiListar`, `apiCriar`, `apiAtualizar`, `apiExcluir`). Migrar do Prompt
Master para um serviço dedicado significa reescrever só essas quatro,
mantendo o resto da página (UI, filtro, import/export) intacto.

## Licença

Veja [LICENSE](LICENSE).
