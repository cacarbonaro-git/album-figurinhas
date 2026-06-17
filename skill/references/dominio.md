# Domínio: figurinhas, coleção e troca

Leia este arquivo antes de modelar dados ou construir telas. Ele descreve as entidades, as regras de negócio e as armadilhas que derrubam projetos de figurinhas. Modelar errado aqui força retrabalho caro depois — o catálogo, o status de cada figurinha e a troca são interdependentes.

## Índice
1. Entidades principais
2. Regras de negócio
3. Matching de troca (conceito)
4. Armadilhas clássicas

---

## 1. Entidades principais

**Álbum (Album / Collection set)**
A coleção completa que se quer preencher. Tem nome, edição/ano, editora (Panini, etc.) e um total de figurinhas. Pode ser fixo e conhecido (a maioria dos álbuns oficiais tem lista pública de figurinhas) ou aberto (usuários cadastram). Decida cedo: catálogo curado pelo admin é mais confiável e permite match preciso; catálogo aberto escala para qualquer álbum mas exige tratar duplicatas e nomes divergentes.

**Figurinha (Sticker / Card)**
Um item do álbum. Atributos típicos: identificador dentro do álbum (raramente um inteiro simples — ver armadilhas), rótulo/nome (jogador, escudo, cidade), grupo/categoria (time, país, raridade), e flags de especial (legend, dourada, brilhante, holográfica). A figurinha pertence a um álbum; é o catálogo, não o que a pessoa possui.

**Usuário / Colecionador (User)**
Quem coleciona e troca. Para multiusuário precisa de identidade (login), e para troca confiável às vezes de reputação e de uma forma de contato mediada. Muitos são menores — trate dados com cuidado (ver princípio LGPD no SKILL.md).

**Coleção do usuário (UserCollection / item de posse)**
A ponte entre Usuário e Figurinha: para cada figurinha do álbum, qual o status daquela pessoa. Modele como uma linha por (usuário, figurinha) com:
- `status`: falta | tenho | repetida
- `quantidade`: quantas cópias tem (0 = falta; 1 = tenho; 2+ = tem repetidas para trocar). Modelar quantidade como inteiro é mais robusto que três flags soltas: faltante = 0, possui = ≥1, disponível para troca = quantidade − 1.

Esta é a tabela que mais cresce (usuários × figurinhas) e a que o matching consulta. Indexe bem.

**Proposta de troca (Trade / TradeProposal)**
Conecta duas coleções: figurinhas que A dá para B e que B dá para A, mais um estado (sugerida, proposta, aceita, combinada, concluída, cancelada). Mesmo que a v1 só *sugira* matches sem gerenciar o ciclo completo, modele a troca como entidade desde o início — embutir isso depois é doloroso.

---

## 2. Regras de negócio

- **Faltante**: figurinha do álbum onde a quantidade do usuário é 0. O conjunto de faltantes é (todas as figurinhas do álbum) − (as que ele tem). Calcular contra o catálogo, não só listar o que foi marcado.
- **Repetida (disponível para troca)**: quantidade ≥ 2. Só a partir da segunda cópia é que sobra para trocar; a primeira ele guarda para colar. Disponível para troca = quantidade − 1.
- **Progresso do álbum**: distintas possuídas ÷ total do álbum. Métrica que o colecionador mais quer ver ("faltam 23 para completar").
- **Troca justa**: idealmente 1 repetida sua por 1 faltante minha, e vice-versa, em quantidades equilibradas. Figurinhas raras/legend valem por várias comuns na prática da comunidade — se modelar raridade, permita troca ponderada, mas não imponha um "preço": deixe os dois combinarem.
- **Raridade/especiais**: legend, douradas e brilhantes são mais difíceis e mais desejadas. Marque-as no catálogo; elas mudam o comportamento de match (gente segura repetida de legend).

---

## 3. Matching de troca (conceito)

O match entre dois usuários A e B é a interseção de dois conjuntos:
- O que A pode dar a B = repetidas de A ∩ faltantes de B
- O que B pode dar a A = repetidas de B ∩ faltantes de A

Uma troca vale a pena quando os dois conjuntos são não-vazios (troca mútua). O sistema deve, para um usuário, varrer os outros e ranquear por quão boa é a troca (quantas figurinhas casam dos dois lados, equilíbrio entre o que cada um dá). O algoritmo concreto e o SQL estão em `arquitetura.md` — aqui o importante é entender que **match é interseção de conjuntos de repetidas e faltantes, e bom match é mútuo e equilibrado**.

Propriedade que sempre deve valer: simetria. Se A aparece como bom par de troca para B, B precisa aparecer para A. Teste isso.

---

## 4. Armadilhas clássicas

- **Numeração não é um inteiro sequencial.** Álbuns usam códigos como "BRA1, BRA2", "FWC", "C1..C32" por time, ou prefixos por categoria. Modele o identificador da figurinha como string/código, não como `int` puro, ou você não consegue cadastrar metade dos álbuns reais.
- **Figurinhas especiais fora da contagem normal.** Legends, escudos brilhantes e extras às vezes têm numeração própria ou paralela. Não assuma "1 até N contíguo".
- **Repetidas em quantidade.** "Tenho 5 da número 10" é comum. Modelar status como booleano (tenho/não tenho) joga fora a informação que faz a troca funcionar. Use quantidade.
- **Listas vêm do WhatsApp em texto.** O dado de entrada real é "tenho: 1, 4, 9, 9, 9 / falta: 12, 30". Saber importar isso (inclusive repetidas pela contagem) é o que faz a ferramenta ser adotada.
- **Catálogo aberto gera duplicatas e nomes divergentes.** Se usuários cadastram figurinhas, "Neymar Jr" e "Neymar" viram itens diferentes e quebram o match. Prefira catálogo curado por álbum; se aberto, normalize e deduplique.
- **Escala da tabela de coleção.** 10 mil usuários × 700 figurinhas = 7 milhões de linhas. O match não pode ser O(usuários²) ingênuo varrendo tudo. Indexe por figurinha e filtre por álbum; veja a estratégia em `arquitetura.md`.
- **Confiança e segurança entre estranhos.** Troca presencial entre desconhecidos, muitos menores. A modelagem de contato/encontro é tão importante quanto a técnica — medie, não exponha endereço, e considere reputação.
