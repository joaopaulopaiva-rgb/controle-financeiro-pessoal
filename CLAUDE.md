# Extrato Vivo — Controle Financeiro Pessoal

Contexto persistente do projeto. Qualquer sessão do Claude Code que trabalhe neste repositório deve ler este arquivo primeiro.

Este projeto é pessoal e **não tem relação** com o repositório `Diretoria-de-Compras` (que é institucional/UFRN) — são domínios separados de propósito.

## 1. O que é este projeto

Controle financeiro pessoal com dois modos de uso:

1. **Lançamento manual contínuo** — a pessoa registra gastos/receitas sempre que lembrar, pelo app.
2. **Análise mensal de extratos** — no fim do mês, a pessoa envia os extratos de cartão e Pix (PDF/CSV, numa conversa com o Claude), e o Claude lê, cruza com os lançamentos manuais do mês e gera uma análise.

## 2. O app (Artifact)

O "aplicativo" é uma página publicada como Claude Artifact, com banco de dados embutido (`db` capability) — funciona como um mini-PWA: abre em qualquer navegador (celular ou computador), sem instalação, e os dados persistem no armazenamento do artifact (não em `localStorage` — sobrevive a reload, troca de aparelho, etc.).

- **Link do app:** https://claude.ai/code/artifact/fb16885b-9529-45ef-9da3-c12bc900594f
- **Código-fonte:** `app/financas.html` — sempre editar este arquivo e republicar (mesmo `file_path`) para atualizar o app, nunca criar um novo artifact para a mesma finalidade (perde a URL e o banco de dados associado).
- **Capabilities declaradas:** `db` apenas. A capability `user` (que permitiria dados privados por visitante via `data/users/me/...`) não estava disponível para esta conta na sessão em que o app foi criado — como o uso é de uma pessoa só, os dados ficam em coleções compartilhadas no nível raiz do banco (não há necessidade prática de isolamento por usuário). Se `user` ficar disponível no futuro e for interessante usar (ex.: compartilhar o app com mais alguém sem misturar dados), reavaliar.
- Artifact é privado por padrão (só quem tem o link/está logado como dono acessa) — não foi compartilhado publicamente.

## 3. Modelo de dados (banco do Artifact)

Coleções no banco do app (ver `db` capability — documentos JSON, até 5.000 no total no artifact):

- **`transacoes`** — uma doc por lançamento. Campos: `tipo` (`"despesa"` | `"receita"`), `valor` (number), `categoria` (string), `metodo` (`"pix"` | `"credito"` | `"debito"` | `"dinheiro"` | `"transferencia"`), `data` (string `"YYYY-MM-DD"`), `descricao` (string, opcional), `origem` (`"manual"` | `"extrato"`), `criado_em` (ISO datetime).
- **`resumos`** — uma doc por mês analisado, id = `"YYYY-MM"`. Escrita pelo Claude ao processar o extrato mensal (seção 5). Campos: `mes` (mesmo valor do id, para permitir `orderBy`), `receitas_total`, `despesas_total`, `saldo`, `por_categoria` (objeto categoria→valor), `destaques` (array de strings — achados relevantes), `divergencias` (array de strings — diferenças entre o extrato e o que já estava lançado manualmente), `gerado_em` (ISO datetime).
- **`config/categorias`** (documento único) — `{despesa: [...], receita: [...]}`. Lista de categorias, editável pelo próprio app (chip "+ nova" grava aqui). Categorias padrão de despesa: Mercado, Restaurante/Delivery, Transporte, Moradia/Contas, Saúde, Educação, Lazer, Assinaturas, Compras, Cuidados pessoais, Pets, Outros. Receita: Salário, Freelance/Extra, Reembolso, Rendimento, Outros.

Para inspecionar/editar o banco fora do app (ex.: escrever uma análise mensal), usar a ação `read_db`/`write_db` da ferramenta Artifact, com a URL do artifact acima.

## 4. Lançamento manual

Feito diretamente no app (aba "Lançar"): valor, tipo, categoria (chips), método, data (default hoje), descrição opcional. Também pode ser feito **conversando com o Claude** ("gastei 50 no mercado no pix hoje") — nesse caso o Claude escreve direto na coleção `transacoes` via `write_db`, com `origem: "manual"` (é lançamento manual mesmo vindo por chat, não confundir com `origem: "extrato"`).

## 5. Análise mensal de extratos

Fluxo quando a pessoa envia o extrato de cartão/Pix (PDF, CSV ou colado em texto) numa conversa:

1. Ler o extrato e extrair os lançamentos (data, descrição, valor).
2. Ler os lançamentos manuais do mês já salvos em `transacoes` (via `read_db`, filtrando `data` no mês).
3. Cruzar: identificar o que já foi lançado manualmente e bate com o extrato, o que está no extrato mas não foi lançado manualmente (adicionar como novo doc em `transacoes` com `origem: "extrato"`, categorizando pelo padrão de categorias existente), e divergências (valor lançado manualmente diferente do extrato, gasto duplicado, etc.).
4. Gerar destaques do mês em texto livre (maiores categorias de gasto, algo fora do padrão, etc.).
5. Escrever o resultado em `resumos/{YYYY-MM}` (`write_db`, `db_op: "set"`) — o app mostra isso automaticamente na aba "Resumo" assim que a doc existir.

**Não versionar os arquivos de extrato brutos no repositório git** (PDFs/CSVs de banco têm dados sensíveis — número de conta, etc.) — processar na conversa e descartar; o que fica persistido é só o resultado estruturado no banco do Artifact (que não é um repositório git público).

## 6. Decisões e convenções

- Moeda: sempre BRL, formatado como `R$ 1.234,56`.
- Datas em `transacoes.data` sempre no formato ISO `YYYY-MM-DD` (permite ordenar como string).
- Sem edição de lançamento existente no app por ora — só criar e excluir. Se precisar corrigir um valor, excluir e relançar.
- Sem multiusuário / permissão — projeto de uma pessoa só.

## 7. Em aberto (ainda não decidido)

- Automação do lembrete de enviar extrato no fim do mês (hoje é sob demanda, a pessoa manda quando lembrar).
- Editar lançamento existente no app (hoje só cria/exclui).
- Gráfico de evolução mês a mês (hoje o Resumo mostra só o mês selecionado).
- Se algum dia o app for compartilhado com mais alguém, revisitar a decisão da seção 2 sobre a capability `user`.
