# Configurar o envio do formulário de CCT

O formulário envia para `/api/cct` (Vercel Function), que manda o e-mail pela **Gmail API** como `contato@siticopmg.org.br` para `cct@siticopmg.org.br`. Não há custo: a Gmail API faz parte do Google Workspace e a conta de serviço não precisa de faturamento.

São quatro etapas. As chaves **nunca** vão para o repositório, mesmo ele sendo privado: ficam só nas variáveis de ambiente da Vercel.

## Estado atual (24/11/2026)

| Peça | Valor |
|---|---|
| Projeto do Google Cloud | `siticop-site`, **só do site** |
| Conta de serviço | `site-formularios@siticop-site.iam.gserviceaccount.com`, sem papéis no Google Cloud |
| Delegação no Admin | Só `gmail.send` |
| Política de chaves no projeto | **Substituída (criação de chave liberada).** Pendente: voltar para *Herdar a política do pai* (ver o fim da etapa 1) |

> **Histórico:** até 24/11/2026 a conta ficava no projeto `migracao-webmail`, junto com a do script de migração de e-mail. Ao apagar esse projeto, a chave do site deixou de valer e o formulário passou a falhar com `invalid_grant`. Por isso o site tem projeto próprio: **não apague o `siticop-site` nem a `site-formularios`**.

## 1. Google Cloud: conta de serviço

1. Acesse [console.cloud.google.com](https://console.cloud.google.com) com uma conta de administrador do Workspace e crie o projeto `siticop-site`, na organização `siticopmg.org.br`.
2. **APIs e serviços → Biblioteca →** procure **Gmail API** → **Ativar**.
3. **IAM e administrador → Contas de serviço → Criar conta de serviço**
   - Nome: `site-formularios`. Não dê nenhum papel: a permissão de envio vem só da delegação da etapa 2. Não use uma conta padrão do projeto.
4. Abra a conta criada:
   - Em **Detalhes**, copie o **ID exclusivo** (um número longo). Ele é usado na etapa 2.
   - Em **Chaves → Adicionar chave → Criar nova chave → JSON**. O arquivo é baixado.

> **Criação de chave bloqueada:** a organização aplica `iam.managed.disableServiceAccountKeyCreation`. Para liberar só neste projeto (exige o papel *Administrador da política da organização* na organização):
>
> 1. Com o projeto `siticop-site` selecionado: **IAM e administrador → Políticas da organização**, filtre por `iam.managed.disableServiceAccountKeyCreation` → **Gerenciar política**.
> 2. **Substituir a política do pai**, aplicação **Substituir**, e **uma única regra** com aplicação **Desativado** (apague a regra *Ativado* se ela aparecer). **Definir política** e esperar um ou dois minutos.
>
> A política só impede **criar** chaves; a chave já criada continua funcionando. Depois de baixar o JSON, volte a política para **Herdar a política do pai**.

## 2. Admin do Workspace: permissão só para enviar

1. Acesse [admin.google.com](https://admin.google.com) → **Segurança → Acesso e controle de dados → Controles de API → Gerenciar delegação em todo o domínio**.
2. **Adicionar novo**:
   - **ID do cliente:** o ID exclusivo copiado acima.
   - **Escopos OAuth:** `https://www.googleapis.com/auth/gmail.send`
3. **Autorizar**.
4. Se houver outra entrada `site-formularios` com ID diferente (de uma chave ou projeto antigos), apague-a.

Com esse escopo a chave só consegue **enviar** e-mail. Ela não lê nenhuma caixa de entrada.

## 3. Cloudflare: Turnstile (antirrobô)

1. No painel da Cloudflare, abra o Turnstile pela **Pesquisa rápida** (Ctrl+K → "Turnstile") ou em **Segurança do aplicativo → Turnstile**, e clique em **Adicionar widget**.
2. Nome: `Site SITICOP`. **Hostnames:** `siticopmg.org.br` (adicione também o domínio `.vercel.app` do projeto se quiser testar nos previews). **Modo:** Gerenciado. **Pré-autorização:** Não.
3. Guarde a **Chave do site** (pública) e a **Chave secreta**.
4. A **Chave do site** fica em `public/index.html` (atributo `data-sitekey`). Já está configurada: `0x4AAAAAAFOVwO0MN_MjOQ4-`. Se o widget for recriado, troque ali.

## 4. Vercel: variáveis de ambiente

Em **Project → Settings → Environment Variables**, ambiente **Production**:

| Nome | Valor |
|---|---|
| `GMAIL_SERVICE_ACCOUNT_KEY` | O conteúdo inteiro do arquivo JSON da etapa 1 (marque como *Sensitive*) |
| `GMAIL_SENDER` | `contato@siticopmg.org.br` |
| `CCT_DESTINATARIO` | `cct@siticopmg.org.br` |
| `TURNSTILE_SECRET_KEY` | A chave secreta da etapa 3 (marque como *Sensitive*) |

Depois: **Deployments → … → Redeploy**. Apague o arquivo JSON do computador depois de colar.

Se configurar também o ambiente **Preview**, os previews enviam e-mails de verdade. Sem `TURNSTILE_SECRET_KEY`, a verificação antirrobô só é dispensada fora de produção.

## Testar

- Envie um pedido pelo site e confira a chegada em `cct@`. O assunto é `Trabalhador - Solicitação de CCT <ano>` ou `Empresa - Solicitação de CCT <ano>`, e "Responder" vai para o e-mail da pessoa.
- Se der erro, veja **Vercel → Logs** filtrando por `[cct]`. Os logs mostram só a causa técnica, nunca os dados da pessoa.
- **No plano Hobby o log dura cerca de 30 minutos.** Se a falha for antiga, peça para repetir o envio e leia o log logo em seguida. A mensagem genérica do site ("Houve um problema ao enviar a mensagem…") aparece para qualquer falha do envio; só o log diz a causa.

| Mensagem no log | Causa provável |
|---|---|
| `GMAIL_SERVICE_ACCOUNT_KEY não configurada` | Variável ausente ou o deploy não foi refeito |
| `Google recusou a autorização (400)` com `invalid_grant` / `No valid verifier for issuer` | A chave, a conta de serviço ou o projeto do Google Cloud foram apagados ou desativados. Refazer as etapas 1, 2 e 4 |
| `Google recusou a autorização (401)` com `unauthorized_client` | Delegação da etapa 2 ausente, com ID errado ou escopo diferente. Delegação nova pode levar alguns minutos para valer |
| `Gmail recusou o envio (400)` com `failedPrecondition` | Gmail API não ativada no projeto |
| `TURNSTILE_SECRET_KEY não configurada` | Variável ausente em produção |
