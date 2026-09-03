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
- **Interface (desde 03/set/2026): "Central de Indicadores"** — não é mais um app de abas (Lançar/Resumo/Extrato); virou um painel único no estilo BI, baseado numa referência visual que a pessoa mandou. Sem rolagem por seção: tudo num grid de widgets, com drill-through entre eles.
  - **Cartão de filtro** (centro, navy): ano (um por vez) + grade dos 12 meses daquele ano (meses sem lançamento ficam desabilitados/apagados). Seleção de mês é **multi-select** — dá pra combinar vários meses no mesmo período. Esse "período selecionado" alimenta os widgets abaixo, exceto os três de evolução histórica.
  - **Meses com maiores despesas** — top 5 meses (histórico completo, não só o ano do filtro), barra horizontal empilhada por categoria. Clicar num mês troca o período do filtro pra aquele mês só e rola até o Extrato.
  - **Evolução das despesas / Evolução do saldo / Evolução da receita** — os três sempre mostram **todo o histórico** (todos os meses com pelo menos um lançamento), independente do período selecionado no filtro — são os únicos widgets que ignoram o filtro, de propósito (mostrar tendência de um mês só não faz sentido). Saldo em linha (aceita negativo), despesas em barra, receita em barra empilhada (salário vs. extra).
  - **Top 5 despesas (subcategoria)** e **Categoria × subcategoria (treemap)** — ambos escopados ao período do filtro. Clicar num item/bloco também seta o filtro do Extrato (categoria, ou categoria+subcategoria) e rola até lá.
  - **Taxa de poupança do período** — medidor (arco), `(receitas − despesas) / receitas` do período selecionado. Não existe "meta" cadastrada no app — é só esse indicador calculado.
  - **Análise do extrato** — só aparece quando exatamente **1 mês** está selecionado no filtro (mesma lógica de antes: lê a coleção `resumos`, mostra destaques/divergências daquele mês). Some quando o período tem 0 ou 2+ meses selecionados.
  - **Extrato completo** — tabela cheia, sempre no fim, respeitando o período do filtro + os próprios filtros de categoria/subcategoria (que também aceitam ser setados por clique nos outros widgets, como no parágrafo acima). Excluir lançamento é o "×" no fim da linha.
  - **Lançar** virou um modal (botão "+ Novo lançamento" no cabeçalho), não mais uma aba/seção fixa — mesmo formulário de sempre (tipo, valor, categoria→subcategoria em chips com "+ nova" pra criar categoria/subcategoria na hora, método, data, descrição).
  - Cabeçalho mantém o "saldo do mês" (sempre o mês corrente real, `isoMonth(new Date())` — independente do período escolhido nos widgets, é só um lembrete rápido de "como estou esse mês").
  - Paleta navy + dourado, tipografia IBM Plex (igual já era). As cores por categoria continuam as mesmas de sempre (`CATEGORY_COLORS`) — só o layout ao redor mudou.
  - Responsivo: abaixo de ~980px de largura o grid vira uma coluna só (empilha os widgets), pensado pra continuar usável no celular mesmo sendo um layout de dashboard denso.
- Carregamento de dados é por busca direta (`get()`), não por escuta ao vivo (`onSnapshot`) — havia um bug real de `onSnapshot` engolindo um erro de mutação em objeto congelado (`d.data()` é read-only; nunca fazer `v.id = d.id` direto nele, sempre `Object.assign({}, d.data(), {id: d.id})`). O app recarrega ao abrir e depois de salvar/excluir; não tem mais botão ↻ manual (o grid inteiro já recalcula sozinho a cada ação).

## 3. Modelo de dados (banco do Artifact)

Coleções no banco do app (ver `db` capability — documentos JSON, até 5.000 no total no artifact):

- **`transacoes`** — uma doc por lançamento. Campos: `tipo` (`"despesa"` | `"receita"`), `valor` (number), `categoria` (string, macro), `subcategoria` (string, só para despesa — ver taxonomia na seção 3.1), `titular` (`"joao"` | `"melina"` | `"sophia"` — de qual cartão/pessoa veio o gasto; opcional, default implícito é João Paulo quando ausente), `metodo` (`"pix"` | `"credito"` | `"debito"` | `"dinheiro"` | `"transferencia"`), `data` (string `"YYYY-MM-DD"`), `descricao` (string, opcional), `origem` (`"manual"` | `"extrato"`), `criado_em` (ISO datetime).
- **`resumos`** — uma doc por mês analisado, id = `"YYYY-MM"`. Escrita pelo Claude ao processar o extrato mensal (seção 5). Campos: `mes` (mesmo valor do id, para permitir `orderBy`), `receitas_total`, `despesas_total`, `saldo`, `por_categoria` (objeto categoria→valor), `destaques` (array de strings — achados relevantes), `divergencias` (array de strings — itens não identificados ou que precisam confirmação), `gerado_em` (ISO datetime).
- **`config/categorias`** (documento único) — `{despesa: {categoria: [subcategorias...]}, receita: [...]}`. Despesa é **dois níveis** (categoria + subcategoria), receita continua só um nível. Editável pelo próprio app (chip "+ nova" grava aqui, tanto categoria quanto subcategoria). Ver taxonomia completa na seção 3.1.
- **`unifan`** — coleção separada, **fora do controle financeiro ativo** (não entra em `transacoes`, não aparece no app, não conta em `resumos`/Extrato). Guarda gastos que a pessoa identifica como "da Unifan" (ver seção 3.3, regra fixa), preservados para uso futuro. Mesmo shape de documento que `transacoes`.

Para inspecionar/editar o banco fora do app (ex.: escrever uma análise mensal), usar a ação `read_db`/`write_db` da ferramenta Artifact, com a URL do artifact acima.

### 3.1 Taxonomia categoria → subcategoria (despesa)

Decisão da pessoa dona do projeto: toda categoria de despesa tem subcategorias (ex.: Transporte → Combustível/Uber/Estacionamento, não só "Transporte" solto). Taxonomia atual (em `config/categorias`, editável a qualquer momento pelo app):

| Categoria | Subcategorias |
|---|---|
| Alimentação | Supermercado, Padaria, Cafeteria, Doces/Salgados, Bebidas, Mercadinho, Feira, Restaurante, Delivery, Almoço no trabalho |
| Transporte | Uber/Corridas, Combustível, Estacionamento, Manutenção, Limpeza |
| Saúde | Plano odontológico, Academia, Farmácia, Médico/Especialista, Atividade física, Pelada de JP, Plano de Saúde |
| Educação | Inglês, Reforço escolar, Escola/Creche, Lanches, Material escolar, Curso/Plataforma |
| Cuidados pessoais | Cabelo, Estética/Unha, Óptica, Perfumaria, Fisioterapeuta |
| Compras | Roupas/Calçados, Móveis/Decoração, Joias, Eletrônicos, Marketplace/Geral, Loja de departamento, Casa/Eletro, Esporte, Diversos |
| Lazer | Passeios/Parques, Eventos, Colecionáveis, Jogos, Passeio, Viagens |
| Assinaturas | Streaming, Jogos, Produtividade, Fidelidade |
| Moradia | Água, Internet/TV, Energia, Aluguel, Jardinagem, Diaristas, Manutenção, Serviços de reparo |
| Outros | Não identificado, Documentos |

Receita (um nível só): Salário, Escola de Pedro. (Freelance/Extra, Reembolso, Rendimento e Outros foram removidos em 03/set/2026 — decisão da pessoa dona do projeto; nenhum lançamento existente usava essas categorias, então a remoção não deixou nada órfão. "Escola de Pedro" é receita mesmo, confirmado — não é a mensalidade dele, que é despesa em Educação > Escola/Creche.)

**Revisão de 03/set/2026** (decisões coletadas via ferramenta própria de revisão, ver conversa) — mudanças em relação à taxonomia original:
- **Alimentação**: "Restaurante/Delivery" foi separado em "Restaurante" e "Delivery"; "Almoço no trabalho" é novo (specific pra almoços em dia de trabalho, ex. Apurn). Os 81 lançamentos que usavam a subcategoria antiga foram reclassificados retroativamente: os 5 da Apurn → Almoço no trabalho; os 4 com "iFood" no nome do comerciante → Delivery; os 63 restantes (nomes de restaurante "normais", sem indicação de app de entrega) → Restaurante. Critério: comerciante de app de delivery aparece com o nome do app na fatura (iFood, Rappi etc.); nome de restaurante "puro" sem esse padrão foi tratado como consumo no local.
- **Transporte**: "Carro/Manutenção" virou só "Manutenção"; "Limpeza" é nova. Os 9 lançamentos antigos: os 2 "LM Lava Jato" → Limpeza; os outros 7 (Sport Ka película, Urentcar, Jucelinho Pneus) → Manutenção (troca direta de nome, mesmo significado).
- **Saúde**: novas "Pelada de JP" (futebol informal do João Paulo) e "Plano de Saúde" (diferente de "Plano odontológico").
- **Cuidados pessoais**: nova "Fisioterapeuta".
- **Lazer**: nova "Viagens".
- **Moradia**: "Condomínio" removida (nenhum lançamento existente usava); novas "Diaristas", "Manutenção" (troca de serviço, não confundir com a de Transporte — mesmo nome, categorias diferentes), "Serviços de reparo".
- **Receita**: reduzida a só "Salário" (ver acima).

Se algum dos itens acima (principalmente a reclassificação em massa de Restaurante vs. Delivery, que foi inferida por mim a partir do nome do comerciante, não confirmada um por um) estiver errado, é só pedir pra eu corrigir — a lista completa do que foi movido está registrada na conversa de 03/set/2026.

## 3.2 Quem é quem (família)

Confirmado em conversa (02/set/2026), a partir da análise das faturas do cartão C6:

- **João Paulo** (dono do projeto) — cartão principal físico e cartão virtual.
- **Melina** — esposa de João Paulo (confirmado: aliança de casamento comprada na Vivara é "de João Paulo e Melina"). Cartão adicional próprio. Tem plano de academia (Wellhub) e assinaturas de IA (Claude.ai, Anthropic, OpenAI, Suno, Gamma.app) de **uso profissional dela** — essas assinaturas de IA ficam de fora do controle financeiro pessoal por decisão explícita.
- **Sophia Saldanha** — filha, **universitária** (confirmado via "Toca Universitária", lanchonete perto da faculdade). Cartão adicional próprio, usado principalmente para lanches (escola/faculdade) quase diários — categorizados em Educação > Lanches, não em Alimentação.
- **Pedro** — filho, adolescente (tem médico hebiatra — especialista em adolescentes — e curso de inglês/professora de reforço). **Não tem cartão próprio**; os gastos dele aparecem nos cartões dos pais/mãe (ex.: mensalidade escolar, hebiatra, inglês, reforço, figurinhas de álbum de Copa).
- Gastos dos 4 entram **juntos no mesmo controle** (decisão explícita: "família — juntar tudo no mesmo controle"), sem separação obrigatória por pessoa — o campo `titular` existe só como metadado de rastreabilidade, não é usado para segregar totais no app hoje.

## 3.3 Regras de importação de fatura de cartão (aprendidas na 1ª rodada — faturas C6 de maio/junho/julho 2026)

- **Cada parcela é um lançamento no mês em que é cobrada** (não o valor total de uma vez no mês da 1ª parcela) — decisão explícita da pessoa, reflete o caixa real.
- **"Inclusão de Pagamento"** (linha que mostra o pagamento da fatura anterior acontecendo) — nunca lançar, não é gasto nem receita.
- **"Estorno"** — nunca lançar como transação própria; se o estorno cancela uma compra do mesmo mês/fatura, a compra original também não entra (net zero). Se o estorno referencia uma compra de fatura anterior fora do escopo sendo processado, apenas ignorar o estorno (não dá pra abater o que não foi lançado).
- **Anuidade do cartão + seu estorno correspondente** (no C6, a anuidade sempre vem com um "Estorno Tarifa" do mesmo valor no mesmo lançamento) — os dois se cancelam, **nenhum dos dois entra no controle**.
- Nas faturas C6, uma compra **parcelada exibe sempre a data da compra original**, igual em todas as faturas subsequentes — **não é a data da cobrança real**. Para o campo `data` de um lançamento parcelado, usar uma data dentro do mês em que aquela fatura específica está cobrando (não o texto literal impresso na linha).
- Item de pessoa física ou empresa não identificada → perguntar à pessoa antes de categorizar (mesmo que pareça óbvio) — ver seção 3.4 para o que já foi resolvido.
- Alguns gastos que aparecem na fatura (mesmo em cartão da família) **não são da família** — a pessoa pode dizer explicitamente "excluir"/"retirar"/"são gastos de outra pessoa"; nesse caso o item não entra em `transacoes` de jeito nenhum.
- **Regra fixa (combinada 02/set/2026): sempre que a pessoa disser que algo "é da Unifan"**, o lançamento não entra (ou sai, se já tiver entrado) de `transacoes`/`resumos`/Extrato — ele é guardado à parte na coleção `unifan` (seção 3, mesmo shape de documento), preservado para uso futuro, mas fora do controle financeiro ativo.

## 3.4 Comerciantes/pessoas já identificados (não perguntar de novo)

- **BMB\*Nosso CEI** (R$ 2.589,84 fixo/mês) — mensalidade escolar/creche → Educação > Escola/Creche.
- **Apurn** — restaurante onde João Paulo almoça em dia de trabalho → Alimentação > Restaurante/Delivery.
- **F C Menezes Lanches** / **Toca Universitária** (cartão da Sophia) → Educação > Lanches.
- **DL\*Uberrides / Uber Uber\*Trip Help** → Transporte > Uber/Corridas.
- **Midway / Midwest** → estacionamento do shopping Midway → Transporte > Estacionamento.
- **U B Comercio Ltda** — loja de departamento (a pessoa disse que no futuro vai detalhar o que compra lá especificamente).
- **Jim.com\*Katia Cristi** — professora de reforço do Pedro → Educação > Reforço escolar. **Jim.com\*Dondocas** — esmalteria da Melina → Cuidados pessoais > Estética/Unha.
- **Genner Barbosado** — médico hebiatra do Pedro → Saúde > Médico/Especialista.
- **Renilda Saldanha** — cabeleireira da Melina → Cuidados pessoais > Cabelo.
- **Simonettis Ensino** — curso de inglês do Pedro → Educação > Inglês.
- **Ricardo Rodrigues** — móveis de área externa → Compras > Móveis/Decoração.
- **Caroline Droguett Lin** — roupa íntima da Melina → Compras > Roupas/Calçados.
- **Vivara** (compra grande parcelada, ~R$4.480 total) — alianças de casamento de João Paulo e Melina → Compras > Joias.
- **Santa Rita Decor** — presente de casa nova da Joyce → Compras > Móveis/Decoração.
- **Kalsi Bem** — calçados da Melina → Compras > Roupas/Calçados.
- **Sport Ka** — película do carro → Transporte > Carro/Manutenção.
- **Banca Souza e Silva / Bancasouzasilva** — figurinhas do álbum da Copa do Pedro → Lazer > Colecionáveis.
- **Água comprada correndo** (Mariaelianebritok, Marialaurasilvade, Joseanefrancisco, Roberto Fernandes Dini — maquininhas avulsas de valor pequeno) → Saúde > Atividade física.
- **Ulisses Henrique Holan** — restaurante Poke → Alimentação > Restaurante/Delivery.
- **Zig\*Teatro Riachuelo / Zig\*Olimpo / Dex** — bar de evento que João Paulo participou → Lazer > Eventos.
- **Jetshr** — passeio de patinete → Lazer > Passeio.
- **Irachai** — restaurante japonês → Alimentação > Restaurante/Delivery.
- **IFD\*...** (prefixo) → iFood → Alimentação > Restaurante/Delivery.
- **Assinaturas de IA no cartão da Melina** (Claude.ai, Anthropic, OpenAI/ChatGPT, Suno, Gamma.app) — uso profissional dela, **excluídas do controle**, nunca lançar.
- **MP\*Kipaopremium ("Ki Pão")** → padaria → Alimentação > Padaria.
- **LC Comercio** = Padaria Bonfim, onde João Paulo almoça com frequência → Alimentação > Restaurante/Delivery (apesar do nome "padaria", é usada como restaurante).
- **Bee\*\*Pagtesouro** — provavelmente renovação de passaporte de João Paulo → Outros > Documentos.
- **Conceito Tabacaria** → figurinhas do álbum da Copa do Pedro (mesmo padrão da Banca Souza e Silva) → Lazer > Colecionáveis.
- **Srxis** — não lembra o que é, mas confirmou que é restaurante → Alimentação > Restaurante/Delivery.
- **MP\*Melimais** (cartão da Melina, recorrente ~R$19,90/mês) — conta da Unifan → **fora do controle** (regra fixa da seção 3.3), guardado na coleção `unifan`, não em `transacoes`.
- **MP\*Produtoslucena** e **MP\*Seunaturalpra** (cartão da Melina) — produtos naturais (castanhas etc.) → Alimentação > **Mercadinho**.
- **MP\*Angelafantasi** (cartão da Melina) — feira de frutas → Alimentação > **Feira** (nova subcategoria).
- **Olho D'Água** — já vinha como "mercadinho" no nome → Alimentação > **Mercadinho**.
- **Feira de Holambra** — não é feira de alimento, é jardinagem → Moradia > **Jardinagem** (nova subcategoria).

**Excluídos do controle (confirmados "não são gastos seus/da família" ou "desconsiderar")**: Anabeatrizde, Francisco / 66.061.005 Francisco, N Deluxo II, Resilienza Negocios LT, DRF Comercio Ltda, Freeway, Alpinas Comercio, Comercial LNR Ltda, Mineiros4u, ZIG\*ECN Bett Educar, Virttus Consultoria (consultoria contratada e depois cancelada), Nayara Variedades (uso da escola, não é gasto do João Paulo).

**Ainda não identificados** (ficam com categoria genérica "Outros > Não identificado" — a pessoa já foi perguntada e não reconheceu; não perguntar de novo, só categorizar se ela mencionar espontaneamente): Arianecostada, Purefit Eventos, Veneza Empreendimentos, Dlknet\*AC Capim Macio, MP\*Paulorobertob, MP\*Elenirfrare, MP\*6348672Alice.

## 4. Lançamento manual

Feito pelo modal "Novo lançamento" (botão no cabeçalho do app, desde o redesenho de 03/set/2026): valor, tipo, categoria→subcategoria (chips, com "+ nova" pra criar categoria/subcategoria na hora), método, data (default hoje), descrição opcional. Também pode ser feito **conversando com o Claude** ("gastei 50 no mercado no pix hoje") — nesse caso o Claude escreve direto na coleção `transacoes` via `write_db`, com `origem: "manual"` (é lançamento manual mesmo vindo por chat, não confundir com `origem: "extrato"`).

### 4.1 Atalho por URL (pra ícone/atalho no celular)

Implementado em 03/set/2026 pra permitir um atalho no celular (iOS Shortcuts, ou equivalente Android) abrir o app já com o lançamento pronto pra confirmar, sem precisar de nenhuma integração externa (WhatsApp foi cogitado e descartado por enquanto — exigiria WhatsApp Business API + servidor de webhook, infra real demais pro que se ganha; ver decisão na conversa de 03/set/2026 se precisar retomar o assunto).

- **`#lancar`** (sem parâmetros) — abre o modal vazio direto ao carregar a página. Serve pra um ícone na tela de início que pula direto pro lançamento, sem passar pelo painel de indicadores.
- **Parâmetros de query** (combináveis com `#lancar` ou sozinhos — qualquer um deles já dispara a abertura automática do modal):
  - `valor` — aceita vírgula ou ponto decimal (`45,90` ou `45.90`).
  - `desc` ou `descricao` — texto livre.
  - `tipo` — `despesa` (default) ou `receita`.
  - `categoria` / `subcategoria` — só pré-seleciona se o nome bater exatamente com uma categoria/subcategoria já cadastrada (senão fica sem categoria selecionada, pra pessoa escolher na hora).
  - `metodo` — `pix` | `credito` | `debito` | `dinheiro` | `transferencia`.
  - `data` — `YYYY-MM-DD` (default hoje).
- Exemplo: `.../artifact/fb16885b.../?valor=45,90&desc=Padaria&tipo=despesa&categoria=Alimenta%C3%A7%C3%A3o&subcategoria=Padaria`.
- A página **nunca salva sozinha** — só pré-preenche o formulário e abre o modal; a pessoa sempre confirma (ajusta categoria se precisar) e toca em "Salvar lançamento". Depois de aplicar os parâmetros, a URL é limpa (`history.replaceState`) pra um refresh manual não reabrir/repreencher com os mesmos dados.
- A construção do atalho em si (app Atalhos do iOS perguntando valor/descrição por voz ou texto e montando o link) fica do lado da pessoa, no celular — não é algo que dá pra configurar por aqui.

## 5. Análise mensal de extratos

Fluxo quando a pessoa envia o extrato de cartão/Pix (PDF, CSV ou colado em texto) numa conversa:

1. Ler o extrato e extrair os lançamentos (data, descrição, valor).
2. Ler os lançamentos manuais do mês já salvos em `transacoes` (via `read_db`, filtrando `data` no mês).
3. Cruzar: identificar o que já foi lançado manualmente e bate com o extrato, o que está no extrato mas não foi lançado manualmente (adicionar como novo doc em `transacoes` com `origem: "extrato"`, categorizando pelo padrão de categorias existente), e divergências (valor lançado manualmente diferente do extrato, gasto duplicado, etc.).
4. Gerar destaques do mês em texto livre (maiores categorias de gasto, algo fora do padrão, etc.).
5. Escrever o resultado em `resumos/{YYYY-MM}` (`write_db`, `db_op: "set"`) — o app mostra isso automaticamente na aba "Resumo" assim que a doc existir.

**Não versionar os arquivos de extrato brutos no repositório git** (PDFs/CSVs de banco têm dados sensíveis — número de conta, etc.) — processar na conversa e descartar; o que fica persistido é só o resultado estruturado no banco do Artifact (que não é um repositório git público).

**Primeira importação (feita em 02/set/2026):** faturas C6 de maio, junho e julho/2026 (cartão principal + virtual de João Paulo, cartão da Sophia, cartão da Melina), 690 lançamentos no total. Nessa primeira rodada, o processo foi bem mais devagar que o fluxo normal descrito acima — a pessoa pediu para eu perguntar o perfil de cada comerciante não óbvio antes de categorizar (ver seção 3.4 para o que já ficou resolvido). Da próxima vez, comerciantes já mapeados na seção 3.4 não precisam de nova pergunta; só perguntar sobre nomes realmente novos.

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
