# siticop-webpage — instruções para agentes

Contexto geral dos sistemas do SITICOP-MG (repositório irmão `../docs`):

@../docs/README.md
@../docs/adr/G05-site-e-cms.md

## Estado atual

Hoje o site é a **página de contingência**: HTML e CSS estáticos em `public/`,
sem build, e uma Vercel Function em `api/cct.js` para o formulário de CCT
(Gmail API + Turnstile). Estrutura e passo a passo em `README.md` e
`docs/configurar-envio.md`.

O site novo (Next.js) e o CMS (Payload, na EC2) ainda não começaram. Quando
começarem, o repositório passa a ter `site/` e `cms/` ([G05](../docs/adr/G05-site-e-cms.md)).

- Branches: `main` (produção) e `develop`.
- Rodar: `npm run dev` (Node 22, sem `npm install`), em http://localhost:3000.

## Regras

- **Preservar a identidade visual:** azul `#023a51`, verde `#2cbc63`, vermelho
  `#ed3237`, fonte Manrope (self-hosted em `public/fonts/`). O público usa
  principalmente o celular.
- A CSP em `vercel.json` é estrita: **sem script ou estilo inline**, e o único
  domínio externo permitido é `challenges.cloudflare.com` (Turnstile). Recurso
  novo de fora exige ajustar a CSP.
- Segredos só nas variáveis de ambiente da Vercel. `.env.local` é local e
  ignorado pelo git.
- Logs das funções registram só a causa técnica, **nunca dados da pessoa** que
  preencheu o formulário.
- Funções que recebem dados pessoais devem rodar na região `gru1`.
- URLs antigas do WordPress têm redirecionamento em `vercel.json`; não
  removê-los.
