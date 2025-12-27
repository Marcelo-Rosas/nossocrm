# RBAC (Permissões) — FlowCRM

Data: 2025-12-14

Este documento define as permissões de **admin** e **vendedor** (single-tenant, multiusuário).

## Papéis

- **admin**: usuário com poderes administrativos (gestão de equipe e configurações do sistema).
- **vendedor**: usuário operacional (CRM do dia a dia).

## Regra simples (baseline)

- O **vendedor só não pode**:
  1) **criar/remover/gerenciar usuários** (e convites) e
  2) **mexer nas configurações do sistema**.

- **Exceção**: o vendedor **pode editar o próprio perfil**.

## Matriz de permissões (Admin vs Vendedor)

> Observação: a UI esconde itens admin-only, mas **a segurança real é server-side** (endpoints retornam `403` quando o usuário não é admin).

### Telas / Menus (UI)

| Área | Admin | Vendedor | Observações |
|---|---:|---:|---|
| CRM (Dashboard / Deals / Contatos / Atividades / Inbox / Boards / Relatórios) | ✅ | ✅ | Operação do dia a dia. |
| Perfil (editar o próprio perfil) | ✅ | ✅ | Exceção explícita: vendedor pode editar o próprio perfil. |
| Settings → Geral → Página inicial | ✅ | ✅ | Preferência pessoal. |
| Settings → Geral → Tags / Campos customizados | ✅ | ❌ | Configuração global (admin-only). |
| Settings → Central de I.A → Ver status (provedor/modelo/chave configurada) | ✅ | ✅ | Vendedor vê somente leitura. |
| Settings → Central de I.A → “IA ativa na organização” (toggle global) | ✅ | ❌ | Vendedor vê o toggle desabilitado. |
| Settings → Central de I.A → Configurar provedor/modelo/chave | ✅ | ❌ | Configuração global (admin-only). |
| Settings → Central de I.A → Funções de IA (feature flags) + editar prompts | ✅ | ❌ | Configuração global (admin-only). |
| Settings → Dados → Estatísticas | ✅ | ✅ | Visível para ambos. |
| Settings → Dados → Zona de Perigo (“Zerar Database”) | ✅ | ❌ | Ação perigosa (admin-only). |
| Settings → Produtos/Serviços | ✅ | ❌ | Aba/rota admin-only. |
| Settings → Integrações (API/Webhooks/MCP) | ✅ | ❌ | Aba/rota admin-only. |
| Settings → Equipe | ✅ | ❌ | Aba/rota admin-only. |
| Boards → Exportar templates | ✅ | ❌ | Ação disponível apenas para admin. |

### Endpoints (server-side)

| Endpoint | Admin | Vendedor | Notas |
|---|---:|---:|---|
| `GET /api/admin/users` | ✅ | ❌ | Lista usuários da organização. |
| `DELETE /api/admin/users/[id]` | ✅ | ❌ | Remove usuário (admin não pode remover a si mesmo). |
| `GET /api/admin/invites` | ✅ | ❌ | Lista convites ativos. |
| `POST /api/admin/invites` | ✅ | ❌ | Cria convite com role `admin` ou `vendedor`. |
| `DELETE /api/admin/invites/[id]` | ✅ | ❌ | Revoga/remove convite. |
| `GET /api/settings/ai` | ✅ | ✅ | Vendedor **não** recebe as chaves brutas (só flags “tem key configurada”). |
| `POST /api/settings/ai` | ✅ | ❌ | Atualiza settings globais da IA (inclui chaves). |
| `GET /api/settings/ai-features` | ✅ | ✅ | Retorna flags + `isAdmin`. |
| `POST /api/settings/ai-features` | ✅ | ❌ | Atualiza feature flags (admin-only). |
| `GET /api/settings/ai-prompts` | ✅ | ❌ | Lista prompts/overrides (admin-only). |
| `POST /api/settings/ai-prompts` | ✅ | ❌ | Cria nova versão do prompt (admin-only). |
| `GET /api/settings/ai-prompts/[key]` | ✅ | ❌ | Lista versões/ativo (admin-only). |
| `DELETE /api/settings/ai-prompts/[key]` | ✅ | ❌ | Desativa override ativo (admin-only). |
| `GET /api/invites/validate?token=...` | ✅ | ✅ | Fluxo de onboarding: valida convite pelo token (não depende de role logado). |
| `POST /api/invites/accept` | ✅ | ✅ | Fluxo de onboarding: aceita convite e cria usuário (token válido). |

## O que é “configuração do sistema”

Exemplos típicos (admin-only):

- Gestão de equipe/convites
- Webhooks/integrações
- Chaves de API “da empresa” (quando globais)
- Ajustes que afetam todos os usuários (ex.: taxonomias globais: tags/campos personalizados quando forem persistidos no backend)
- Ações perigosas de manutenção (ex.: “zerar database”)

## Preferências pessoais (permitido ao vendedor)

Exemplos típicos (self-service):

- Editar o **próprio** nome, apelido, avatar, etc.
- Preferências de UX (ex.: página inicial)
- Configurações pessoais do assistente/IA (quando são por usuário)

## Regras de implementação (defesa em profundidade)

1) **UI/Rotas**: esconder/limitar seções admin-only no frontend (não é segurança, mas melhora UX e reduz erro).
2) **Server-side**: toda ação admin-only deve ser validada no servidor (Route Handlers / Server Actions) usando o `profile.role` derivado do usuário autenticado.
3) **Banco/RLS**: tabelas sensíveis devem ter políticas que bloqueiem escrita/leitura indevida. 
4) **Service role**: onde for necessário bypass de RLS, aplicar checagem de role ANTES de qualquer operação.

## Check rápido de conformidade

- [ ] Vendor não acessa gestão de usuários
- [ ] Vendor não altera configurações globais (webhooks/chaves/integrations)
- [ ] Vendor consegue editar o próprio perfil
- [ ] Endpoints/Server Actions fazem validação server-side
- [ ] Políticas RLS revisadas para tabelas sensíveis
