# Álbum de Figurinhas — App de coleção e troca

Plataforma para colecionadores de figurinhas catalogarem a coleção e **trocarem entre si**, pensada para crianças e pais, com comunicação pelo WhatsApp. Este repositório contém os protótipos navegáveis e a skill de arquitetura que guia o desenvolvimento.

## O que tem aqui

### `prototipos/`
Protótipos auto-contidos (um arquivo `.html` cada — abrem no navegador e no celular, sem servidor):

- **`prototipo-kids.html`** — o app principal, com cara infantil. Telas: login por WhatsApp, início com mascote e progresso, "Meu álbum" (marcar tenho/falta/repetida), "Trocar" (exporta sua lista e cruza a lista colada do amigo) e "Eu" (escolher álbum por busca de nome, mascote, conta). Coleção por álbum, salva no aparelho.
- **`prototipo-figurinhas.html`** — protótipo de consulta/navegação do catálogo, multi-álbum.
- **`conferencia-troca.html`** — ferramenta enxuta de conferência: cola duas listas e cruza repetidas × faltantes.

### `skill/`
A skill **`arquiteto-album-figurinhas`** — um arquiteto de software sênior que projeta e implementa o produto. Inclui o domínio (entidades, regras, armadilhas de figurinhas/coleção/troca) e a arquitetura de referência (stack web, esquema de banco, algoritmo de matching, deploy).

## Conceitos centrais

- **Modelo de dados**: figurinha com código em *string* (ex.: `BRA1`, `FWC3`), posse com *quantidade* (0 = falta, 1 = tenho, 2+ = repetida).
- **Match de troca** = interseção de conjuntos: você recebe = (suas faltas ∩ repetidas do amigo); você dá = (suas repetidas ∩ faltas do amigo). Simétrico.
- **Definição de álbum**: a pessoa digita o nome e o app busca a referência. Hoje a base é embutida (`ALBUMS_REF`); no produto real vira uma API de catálogo (curada e/ou Colnect).
- **Login**: por número de WhatsApp (OTP simulado no protótipo).

## Status

Protótipos para validação de fluxo e arquitetura. A evolução natural é trocar o armazenamento local (`localStorage`) e a base embutida por um backend leve com login real e catálogo, mantendo a mesma interface.
