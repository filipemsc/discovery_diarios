# discovery_diarios

Repositório de **prototipagem** para um projeto maior de coleta e estruturação de diários
oficiais brasileiros. Aqui testamos, em uma amostra pequena e heterogênea, o pipeline que
depois será generalizado para rodar em escala (todas as esferas, todos os entes).

## Objetivo do projeto maior

Diários oficiais brasileiros são publicados por uma enorme variedade de entes:

- **Federal**: Diário Oficial da União (DOU), seções 1/2/3.
- **Estadual**: Diário Oficial de cada UF (Poder Executivo estadual).
- **Municipal**: Diário Oficial de municípios (quando têm diário próprio) ou de
  **associações/consórcios de municípios** que publicam um diário consolidado para vários
  municípios ao mesmo tempo (ex.: AMM, APM/diariomunicipal.com.br).
- **Judiciário**: diários de tribunais (TJ, TRFs, etc.).
- **Tribunais de Contas**: TCEs e TCMs têm diário oficial eletrônico próprio.
- **Legislativo**: Câmaras Municipais, Assembleias Legislativas, Câmara dos Deputados,
  Senado.
- **Outros órgãos**: autarquias, fundações, empresas públicas, conselhos profissionais
  (CRM, CRC, CREF etc.) frequentemente publicam seus próprios atos dentro do diário do
  ente ao qual estão vinculados.

O objetivo final é um pipeline de coleta + processamento capaz de transformar qualquer um
desses diários em uma base de dados estruturada e padronizada, com granularidade de **ato**
(cada linha = um ato administrativo/normativo publicado), para permitir análise em escala
sobre a administração pública brasileira.

Este repositório testa esse pipeline numa amostra pequena antes de generalizar.

## Estado atual do repositório

```
data/
  raw/     13 PDFs de diários oficiais (amostra heterogênea — ver abaixo)
  int/     (vazio) destino do texto intermediário extraído e limpo
  final/   (vazio) destino da base de dados final estruturada por ato
```

Ainda não há código de processamento neste repositório — este CLAUDE.md define o
contrato de dados e o plano antes da implementação.

### Amostra em `data/raw/` (13 diários)

| Arquivo | Ente / fonte | Esfera | Observações de layout |
|---|---|---|---|
| `2025_12_30_ASSINADO_do1.pdf` | DOU, Seção 1 | Federal | 1 coluna, cabeçalho/rodapé padronizados, sumário com link de seções, ANEXOs tabulares extensos, atos de conselhos profissionais no fim |
| `2026-01-05_ed-3.pdf` | DOM São Paulo (capital) | Municipal | 1 coluna, numeração "Decreto \| Documento: ####", listas tabulares de servidores (matrícula/nome/datas), editais e erratas |
| `2026-01-05_ed-4090.pdf` | DOM Florianópolis (sistema tipo "e-Gestão"/processo) | Municipal | Predominantemente **tabular/orçamentário** (quadros de despesa por funcional-programática), atos embutidos como anexos de processo, assinatura digital por página |
| `2026-01-07_ed-7321.pdf` | Diário Oficial Eletrônico de Maceió/AL | Municipal | 1 coluna, bloco de expediente extenso no início, padrão `ATOS E DESPACHOS DO PREFEITO`, rodapé com `Publicado por:` + `Código Identificador:` (padrão comum a diários do sistema "Diário Oficial dos Municípios" / consórcios) |
| `2026-01-07_ed-8695.pdf` | DOM Eletrônico de Goiânia/GO | Municipal | 1 coluna, cabeçalho com bookmarks de PDF por seção, publica também atos de SEI (termos de apostilamento), avisos de contratação direta extensos (Lei 14.133) |
| `2026-01-08_ed-51260.pdf` | Diário Oficial do Estado de Alagoas (Suplemento) | Estadual | 1 coluna, decretos com anexos orçamentários tabulares (erratas "onde se lê / leia-se"), publica também ato de **consórcio interestadual** (Consórcio Nordeste) dentro do diário estadual |
| `2026-01-09_ed-001_edicao-extra-a.pdf` | Diário Oficial do Distrito Federal (DODF), Edição Extra | Estadual/DF | 3 seções (I/II/III) no sumário, 1 coluna no corpo, portarias conjuntas entre secretarias |
| `2026-01-09_ed-3526.pdf` | Diário Oficial dos Municípios do Estado de São Paulo (APM) | Municipal (consórcio) | Diário **consolidado multi-município** publicado por associação de municípios — cada bloco de atos é precedido pelo nome do município (`ESTADO DE SÃO PAULO / MUNICÍPIO DE X`) |
| `2026-01-09_ed-35620.pdf` | Diário Oficial do Estado do Amazonas | Estadual | 1 coluna, abre com matérias jornalísticas/institucionais (não são atos), marcadores internos `<#E.G.B#...#...>` delimitando cada publicação — **útil como delimitador estrutural de atos** |
| `2026-01-13_ed-1754.pdf` | Diário Eletrônico do TCE de Roraima (DETCERR) | Judiciário/Controle (Tribunal de Contas) | 1 coluna, predominância de Certidões de Quitação e Acórdãos, seção "JURISDICIONAL" |
| `2026-01-13_ed-2932.pdf` | Diário Oficial Eletrônico do TCE do Maranhão | Judiciário/Controle (Tribunal de Contas) | 1 coluna, sumário com múltiplos pontos de preenchimento (dot leaders) que geram ruído na extração de texto, Editais de Citação e Portarias de RH internas do tribunal |
| `2026-01-17_ed-7422.pdf` | Diário Oficial do Município de Belo Horizonte (DOM-BH) | Municipal | 1 coluna, mistura Poder Executivo e **Poder Legislativo** (Câmara Municipal) no mesmo diário, anexos tabulares de cargos/pontos, listas de resultado de concurso público |
| `2026-02-06_ed-15365.pdf` | Diário Oficial do Município de Belém/PA | Municipal | 1 coluna, mistura Executivo e Legislativo (composição da Câmara), inclui **lista de multas de trânsito** (um "ato" por infração/placa — ver seção de tipos de ato) |

Pontos-chave de heterogeneidade a tratar no pipeline:

- **Número de colunas**: a maioria da amostra é 1 coluna, mas o pipeline final deve
  detectar e tratar diários em 2+ colunas (comum em diários estaduais maiores e no DOU
  em outras seções) para não embaralhar a ordem de leitura.
- **Cabeçalho/rodapé repetidos em toda página**: nome do diário, data, número de página,
  texto de certificação digital ("Documento assinado digitalmente..."), hashes/QR code,
  URLs de validação. Devem ser identificados e removidos do texto de cada página (tendem
  a se repetir quase identicamente entre páginas do mesmo diário).
- **Sumário/índice paginado**: presente em quase todos os diários, não é um ato em si,
  mas pode ajudar a identificar os órgãos e a estrutura do diário.
- **Tabelas e anexos financeiros/orçamentários**: comuns em decretos de crédito
  suplementar, PDGs de estatais, quadros de detalhamento de despesa. Podem ser mantidos
  como texto corrido dentro do ato, mas quebram heurísticas de "parágrafo = texto corrido".
- **Listas tabulares de pessoas** (servidores, concursados, multados, intimados): às vezes
  constituem o próprio corpo do ato (ex.: editais de intimação com tabela de CPF/nome),
  às vezes são anexos de um ato maior.
- **Marcadores internos de sistema**: alguns diários (ex.: Amazonas) inserem tags como
  `<#E.G.B#256260#6#259805>` ao redor de cada publicação — uma pista valiosa para
  segmentação automática de atos quando presentes, mas não deve ser assumida como
  universal.
- **Assinatura digital por documento/processo** (padrão SEI, usado por Florianópolis e
  Goiânia): o "ato" pode estar embutido como anexo de um processo administrativo maior,
  com página de assinaturas separada ao final.
- **Diários consolidados multi-ente**: diários de associações/consórcios de municípios
  (ex.: APM-SP, AMM) publicam atos de dezenas de municípios diferentes na mesma edição —
  a identificação do município correto por ato é crítica e normalmente aparece como
  cabeçalho de seção (`MUNICÍPIO DE X`) antes do bloco de atos daquele município.
  Atos municipais também podem aparecer em diários estaduais.
- **Conteúdo não normativo**: editoriais, notícias institucionais, propaganda de serviços
  públicos (campanhas de vacinação, apps), expediente/composição de cargos — não geram
  atos e devem ser descartados ou marcados como `não_ato` na etapa de classificação.

## Pipeline planejado

### Etapa 1 — `data/raw/` → `data/int/` (extração e limpeza)

Converter cada PDF em texto plano organizado, preservando a ordem de leitura correta.

Para cada diário, gerar em `data/int/<nome_do_diario>/`:
- Texto por página (ou por ato, se a segmentação já for possível nesta etapa) após:
  - Detecção e remoção de cabeçalho/rodapé repetido.
  - Detecção de colunas e reconstituição da ordem de leitura correta (não concatenar
    colunas lado a lado como se fossem uma linha só).
  - Tratamento de figuras/imagens (ignorar, ou no mínimo não deixar lixo de OCR/placeholder
    no texto; nenhum PDF da amostra parece ser escaneado/imagem pura, mas o pipeline final
    deve estar preparado para isso com OCR).
  - Tabelas: extrair preservando a associação linha-a-linha de rótulo/valor quando possível
    (evitar que vírgulas/quebras de linha dentro de tabela sejam confundidas com fim de
    frase ou fim de ato).
- Um metadado por diário com: arquivo original, nº de páginas, data de
  publicação, edição, formato detectado (colunas, se tem bookmarks/sumário navegável).

### Etapa 2 — `data/int/` → `data/final/` (estruturação por ato)

Segmentar o texto de cada diário em **atos individuais** e produzir uma base de dados
única e padronizada (ex.: parquet/csv) em `data/final/`, com uma linha por ato.

#### Esquema de colunas proposto

| Coluna | Descrição |
|---|---|
| `id_ato` | Identificador único gerado (ex.: hash ou `{arquivo}_{sequencial}`) |
| `arquivo_original` | Nome do arquivo PDF em `data/raw/` |
| `data_publicacao` | Data de publicação do diário (ISO 8601) |
| `edicao` | Número da edição, conforme consta no diário |
| `ind_extra` | Indicador (0/1) que indica se é uma edição extra ou não |
| `esfera` | `federal` \| `estadual` \| `municipal` \| `distrital` (DF) |
| `poder` | `executivo` \| `legislativo` \| `judiciario` \| `controle_externo` (TCs) \| `mp` (Ministério Público, quando aplicável) |
| `uf` | Sigla da UF |
| `municipio` | Nome do município (quando aplicável; nulo para diários estaduais/federais) |
| `ente_consolidado` | Nome da associação/consórcio quando o diário for multi-município (ex.: "APM", "AMM"); nulo quando o diário é de ente único |
| `orgao` | Órgão/entidade emissor do ato (ex.: "Secretaria Municipal de Educação", "Ministério da Fazenda", "TCE-RR") |
| `tipo_ato` | Categoria normalizada do ato (ver taxonomia abaixo) |
| `numero_ato` | Número/identificador do ato como publicado (ex.: "64.893", "072") |
| `data_ato` | Data de assinatura/emissão do ato, se distinta da data de publicação |
| `ementa` | Ementa/resumo do ato, quando presente no próprio texto |
| `texto_ato` | Texto integral do ato |
| `pagina_inicial` | Página do PDF original onde o ato começa |
| `pagina_final` | Página do PDF original onde o ato termina |
| `assinantes` | Nome(s) e cargo(s) de quem assina o ato (lista) |

Esse esquema deve ser tratado como ponto de partida: ajustar conforme os primeiros testes
de segmentação revelarem necessidades reais (ex.: separar `numero_processo` como coluna própria se
apareceu em quase todos os diários e pode valer uma coluna dedicada).

## Taxonomia de tipos de ato (`tipo_ato`)

Lista consolidada a partir da amostra de 13 diários, para orientar o classificador.
Lista "viva": ao processar novos diários, tipos novos devem ser adicionados aqui.

### Atos normativos (força de lei/regulamento)
- `lei` — inclui leis ordinárias, complementares e delegadas
- `medida_provisoria`
- `decreto`
- `decreto_legislativo`
- `portaria`
- `portaria_conjunta` — emitida por duas ou mais autoridades/secretarias
- `resolucao`
- `instrucao_normativa`
- `ato_normativo_generico` — "Ato GP nº X", "Ato SLU nº X" e variantes específicas de
  cada ente que fazem o papel de portaria mas usam nomenclatura própria

### Atos de pessoal / RH
- `nomeacao`
- `exoneracao`
- `designacao` (inclui substituições temporárias de cargo/função)
- `dispensa`
- `concessao_licenca` (licença-prêmio, licença médica, etc.)
- `concessao_ferias` / `alteracao_ferias` / `suspensao_ferias`
- `concessao_diarias_passagens`
- `progressao_funcional`
- `resultado_concurso_publico` / `convocacao_concurso` / `homologacao_concurso`
- `gratificacao` (concessão/alteração de gratificação)

### Atos de contratos e licitações
- `aviso_licitacao` (inclui aviso de contratação direta, dispensa, inexigibilidade)
- `aviso_suspensao_licitacao`
- `edital_licitacao`
- `homologacao_licitacao`
- `extrato_contrato`
- `extrato_termo_aditivo`
- `extrato_termo_apostilamento`
- `extrato_termo_rescisao`
- `extrato_convenio` / `extrato_termo_fomento` / `extrato_termo_colaboracao` (parcerias
  com OSC sob Lei 13.019/2014)
- `acordo_cooperacao_tecnica`
- `registro_cadastral` (ex.: SUCAF — inscrição/renovação de fornecedor)

### Atos financeiros/orçamentários
- `credito_suplementar` / `abertura_credito_adicional`
- `demonstrativo_orcamentario` (anexos de PDG, quadros de despesa por função/programa —
  frequentemente aparecem como anexo de decreto, não como ato autônomo)

### Atos processuais / jurisdicionais (Judiciário e Tribunais de Contas)
- `edital_citacao`
- `edital_intimacao`
- `certidao_quitacao`
- `acordao`
- `despacho`
- `decisao`

### Comunicações e avisos diversos
- `aviso` — categoria residual para avisos que não se encaixam nas específicas acima
- `comunicado`
- `convocacao` (não ligada a concurso, ex.: convocação de reunião)
- `errata` / `retificacao` (inclui "onde se lê / leia-se" e "republicado por incorreção")
- `extrato_licenca_ambiental`
- `embargo` / `interdicao` (ex.: embargo de funcionamento por fiscalização municipal)
- `multa_infracao` — um registro por infração/placa em listas de infrações de trânsito
  (cada linha da lista tabular deve ser tratada como um ato individual, não a lista inteira
  como um ato só)
- `consulta_publica`

### Não-ato (deve ser filtrado ou marcado, não classificado como ato)
- `sumario_indice`
- `expediente` (composição de cargos/diretoria do próprio diário ou do ente)
- `materia_institucional` (notícia, editorial, campanha de serviço público)
- `demonstrativo_anexo_sem_ato_proprio` (quando o anexo é parte de outro ato já capturado)

> Observação: vários entes usam nomenclatura própria para o que é essencialmente uma
> portaria ou decreto (ex.: "Ato GP", "Ato SLU" em Belo Horizonte). A classificação deve
> mapear esses nomes específicos do ente para a categoria normalizada mais próxima, e
> manter o texto original do rótulo em `numero_ato`/metadado, não descartá-lo.

## Convenções de trabalho neste repositório

- Este é um repositório de **teste/protótipo**. Não assumir que soluções aqui precisam ser
  production-ready; o objetivo é validar abordagens antes de generalizar para o projeto
  maior (que vive em outro lugar).
- Preservar sempre a referência ao PDF original (nome do arquivo + página) em qualquer
  estrutura intermediária ou final, para permitir auditoria manual rápida.
- Novos tipos de ato encontrados ao expandir a amostra devem ser adicionados à taxonomia
  acima, não apenas ao código — a lista deste arquivo é a fonte de verdade da taxonomia.
  
