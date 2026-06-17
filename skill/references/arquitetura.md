# Arquitetura: stack, banco, matching e deploy

Leia ao construir uma plataforma multiusuário de figurinhas. Traz a stack recomendada, o esquema de banco pronto para o domínio, o algoritmo de matching de troca e caminhos de deploy baratos. Adapte ao que o usuário já conhece — se ele já domina outra stack equivalente, use a dele e explique trade-offs.

## Índice
1. Stack recomendada
2. Esquema de banco de dados
3. API mínima
4. Algoritmo de matching de troca
5. Import/export formato WhatsApp
6. Deploy e custo
7. Evolução por escala

---

## 1. Stack recomendada

Padrão (bom equilíbrio entre simplicidade e poder, tiers gratuitos generosos):

- **Backend + banco**: Supabase (Postgres gerenciado + auth + API REST automática) OU um backend leve próprio (FastAPI/Python ou Node/Express) sobre Postgres/SQLite. Supabase encurta muito o caminho porque já entrega login e API sobre as tabelas — ótimo quando quem mantém é leigo.
- **Front**: site estático ou SPA leve. Para mobile-first sem loja de apps, um PWA instalável (HTML/JS + manifest + service worker) cobre 90% dos casos. React/Vite se o projeto crescer; HTML+JS puro se for enxuto.
- **Auth**: e-mail/senha ou link mágico já bastam. Evite pedir dados sensíveis no cadastro.

Regra: escolha o que dá deploy num clique e tier gratuito. Hobby não sustenta infra cara.

## 2. Esquema de banco de dados

Postgres (ajuste tipos para SQLite se for local). Este esquema reflete o domínio em `dominio.md`:

```sql
-- Catálogo (curado por álbum)
CREATE TABLE albums (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL,
  edition     TEXT,            -- ano/edição
  publisher   TEXT,            -- Panini, etc.
  total_count INTEGER NOT NULL
);

CREATE TABLE stickers (
  id        SERIAL PRIMARY KEY,
  album_id  INTEGER NOT NULL REFERENCES albums(id),
  code      TEXT NOT NULL,     -- "10", "BRA1", "FWC3" — string, NÃO int
  label     TEXT,              -- nome do jogador/escudo
  category  TEXT,              -- time/país/grupo
  is_special BOOLEAN DEFAULT FALSE,  -- legend/dourada/brilhante
  UNIQUE (album_id, code)
);

CREATE TABLE users (
  id         SERIAL PRIMARY KEY,
  display_name TEXT NOT NULL,
  city       TEXT,             -- região aproximada para troca; NUNCA endereço
  created_at TIMESTAMPTZ DEFAULT now()
);

-- A ponte: uma linha por (usuário, figurinha) que ele já tocou.
-- quantity 0 = falta explícita; 1 = tem; 2+ = tem repetidas.
-- Faltantes "implícitas" = stickers do álbum sem linha aqui (tratar = 0).
CREATE TABLE user_stickers (
  user_id    INTEGER NOT NULL REFERENCES users(id),
  sticker_id INTEGER NOT NULL REFERENCES stickers(id),
  quantity   INTEGER NOT NULL DEFAULT 0 CHECK (quantity >= 0),
  PRIMARY KEY (user_id, sticker_id)
);
CREATE INDEX idx_user_stickers_sticker ON user_stickers(sticker_id);
CREATE INDEX idx_user_stickers_user    ON user_stickers(user_id);

-- Trocas como entidade desde o v1, mesmo que comece só sugerindo.
CREATE TABLE trades (
  id          SERIAL PRIMARY KEY,
  user_a      INTEGER NOT NULL REFERENCES users(id),
  user_b      INTEGER NOT NULL REFERENCES users(id),
  status      TEXT NOT NULL DEFAULT 'sugerida', -- sugerida|proposta|aceita|concluida|cancelada
  created_at  TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE trade_items (
  trade_id    INTEGER NOT NULL REFERENCES trades(id),
  sticker_id  INTEGER NOT NULL REFERENCES stickers(id),
  from_user   INTEGER NOT NULL REFERENCES users(id), -- quem entrega esta figurinha
  PRIMARY KEY (trade_id, sticker_id, from_user)
);
```

Derivações (não armazene, calcule):
- repetidas de U = `quantity - 1` onde `quantity >= 2`
- faltantes de U = stickers do álbum com quantity 0 ou sem linha
- progresso = distintas com `quantity >= 1` ÷ `albums.total_count`

## 3. API mínima

Endpoints que entregam valor na primeira iteração:
- `POST /collections/:user/import` — recebe texto/lista e popula user_stickers (ver seção 5)
- `GET  /collections/:user/missing` — faltantes
- `GET  /collections/:user/duplicates` — repetidas (quantity ≥ 2)
- `GET  /matches/:user` — usuários com boa troca mútua, ranqueados (ver seção 4)
- `POST /trades` / `PATCH /trades/:id` — criar e mover estado da troca

## 4. Algoritmo de matching de troca

Match entre o usuário U e cada outro usuário O dentro do mesmo álbum:
- `da_para_O`  = repetidas(U) ∩ faltantes(O)
- `recebe_de_O`= repetidas(O) ∩ faltantes(U)
- Troca mútua útil quando ambos não-vazios. Ranqueie por `min(|da_para_O|, |recebe_de_O|)` (gargalo da troca equilibrada) e desempate por total.

Não faça O(usuários²) varrendo tudo. Faça centrado em figurinha e em SQL:

```sql
-- Candidatos a receber as repetidas de U (eles têm falta do que U sobra)
WITH minhas_repetidas AS (
  SELECT sticker_id FROM user_stickers WHERE user_id = :u AND quantity >= 2
),
minhas_faltas AS (
  SELECT s.id AS sticker_id FROM stickers s
  LEFT JOIN user_stickers us ON us.sticker_id = s.id AND us.user_id = :u
  WHERE s.album_id = :album AND COALESCE(us.quantity,0) = 0
)
SELECT o.user_id,
       COUNT(*) FILTER (WHERE o.sticker_id IN (SELECT sticker_id FROM minhas_repetidas)) AS eu_dou,
       COUNT(*) FILTER (WHERE o.sticker_id IN (SELECT sticker_id FROM minhas_faltas)
                        AND o.quantity >= 2)                                            AS eu_recebo
FROM user_stickers o
WHERE o.user_id <> :u
GROUP BY o.user_id
HAVING COUNT(*) FILTER (WHERE o.sticker_id IN (SELECT sticker_id FROM minhas_repetidas) AND o.quantity = 0) > 0
   AND COUNT(*) FILTER (WHERE o.sticker_id IN (SELECT sticker_id FROM minhas_faltas) AND o.quantity >= 2) > 0
ORDER BY LEAST(eu_dou, eu_recebo) DESC;
```

(Esse SQL é um ponto de partida — valide e ajuste os filtros ao seu esquema.) Garanta a **simetria**: o par (A,B) deve render o mesmo conjunto visto de qualquer lado. Teste com dois usuários cujas listas se cruzam parcialmente.

## 5. Import/export formato WhatsApp

A entrada real do colecionador é texto colado. Aceite formatos como:
```
tenho: 1, 4, 9, 9, 30, BRA2
falta: 12, 30, FWC1
```
Parse: separe por seção, normalize separadores (vírgula, espaço, ponto-e-vírgula), conte repetições para quantidade (9,9,9 → quantity 3), case-insensitive para códigos com letra. Na exportação, gere de volta nesse mesmo formato para a pessoa colar no grupo. Esse ida-e-volta é o que faz a ferramenta ser adotada por uma comunidade que já vive no WhatsApp.

## 6. Deploy e custo

- Supabase: tier gratuito cobre comunidades pequenas/médias; banco, auth e API inclusos.
- Front estático/PWA: Netlify, Vercel ou Cloudflare Pages — deploy por git push, grátis.
- Backend próprio (se usado): Fly.io, Railway ou Render têm tiers iniciais baratos.
Documente para o mantenedor: como subir, onde ficam as chaves, e que o tier gratuito tem limites — explique quando vai precisar pagar.

## 7. Evolução por escala

- **30 pessoas (grupo/escola)**: até um app de página única com export/import de texto resolve; backend opcional.
- **Centenas**: Supabase + PWA, matching em SQL como acima, índices em `user_stickers`.
- **Milhares+**: cacheie faltantes/repetidas por usuário, pré-compute matches em background, particione por álbum/região, e adicione reputação e moderação. Não construa isso no dia 1 — projete o modelo para permitir, mas entregue o simples primeiro.
