UC-A04: Bloqueio e Desbloqueio de Conta
Utilizadores: Administrador (Admin)
Objetivo:
Proteger as contas contra ataques de força bruta através de bloqueio automático após tentativas falhadas de login e permitir o desbloqueio controlado exclusivamente por um Administrador, com registo completo de auditoria.

podes detalhar este requisito tecnico:
MarkdownUC-A02: Recuperação de Password
Users: Admin
Fluxo seguro de reset via email com token temporário de uso único.

Fluxo principal
1. Utilizador clica 'Esqueci a password'
→
2. Sistema gera token único (válido 15min)
→
3. Email enviado com link de reset
→
4. Utilizador define nova password
→
5. Token invalidado após uso

Critérios de aceitação
Token expira em 15 minutos
Token de uso único (invalidado após primeiro acesso)
Nova password exige: 8+ chars, maiúscula, número, especial
Notificação de confirmação após reset

Link de reset só funciona uma vez. Tentativa de reutilização deve gerar alerta de segurança.Sim, claro! Aqui tens uma versão mais detalhada, clara e profissional deste requisito técnico UC-A02: Recuperação de Password.
UC-A02: Recuperação de Password
Utilizadores: Administrador (Admin)
Objetivo: Permitir que o utilizador Admin recupere o acesso à sua conta de forma segura através de um processo de reset de password via email, utilizando um token temporário de uso único.

Fluxo Principal (Happy Path)

O utilizador Admin acede à página de login e clica no link "Esqueci a password".
O sistema solicita o email associado à conta.
O sistema valida se o email existe na base de dados.
O sistema gera um token de reset:
Único (UUID v4 ou equivalente criptograficamente seguro)
Válido por 15 minutos
Armazenado na base de dados associado ao utilizador (com data de expiração e flag used = false)

O sistema envia um email para o utilizador contendo:
Um link de reset seguro (ex: https://app.dominio.com/reset-password?token=abc123...)
Tempo de validade do link (15 minutos)
Aviso para não partilhar o link

O utilizador clica no link recebido no email.
O sistema valida o token:
Verifica se existe, não está expirado e ainda não foi utilizado.

Se válido, o sistema apresenta o formulário para definir nova password.
O utilizador introduz a nova password e confirma-a.
O sistema valida os requisitos de complexidade da password.
O sistema atualiza a password do utilizador (com hash seguro — recomendado: Argon2id ou bcrypt).
O sistema invalida imediatamente o token (used = true ou elimina o registo).
O sistema envia uma notificação de confirmação por email ao utilizador informando que a password foi alterada com sucesso.
O utilizador é redirecionado para a página de login.


Critérios de Aceitação (Acceptance Criteria)
Segurança do Token

O token deve ter validade máxima de 15 minutos a partir do momento da sua geração.
O token é de uso único: após o primeiro acesso bem-sucedido ao link, deve ser imediatamente invalidado.
Tentativa de reutilização do mesmo token (mesmo que ainda dentro do prazo de 15min) deve ser rejeitada.
Qualquer tentativa de reutilização de token inválido/expirado/usado deve registar um alerta de segurança no sistema (log de segurança + eventual notificação ao admin ou equipa de segurança).

Requisitos da Nova Password
A nova password deve cumprir os seguintes critérios mínimos:

Mínimo de 8 caracteres
Pelo menos 1 letra maiúscula (A-Z)
Pelo menos 1 número (0-9)
Pelo menos 1 caractere especial (! @ # $ % ^ & * ( ) - _ = + [ ] { } | ; : ' " , . < > / ?)

Outros Critérios

O link de reset deve conter o token como parâmetro de query string de forma segura.
O sistema deve impedir ataques de rate limiting na funcionalidade de "Esqueci a password" (ex: máximo 3 tentativas por email por hora).
Todas as operações sensíveis (geração de token, validação, alteração de password) devem ser devidamente registadas em logs de auditoria.
A password deve ser armazenada sempre com hash + salt (nunca em texto plano).
Após o reset bem-sucedido, o utilizador deve receber um email de confirmação com informações como:
Data e hora da alteração
Recomendação para alterar a password regularmente
Alerta caso não tenha sido ele a efetuar o reset (para detetar possíveis compromissos)



Fluxos Alternativos / Exceções

Email não encontrado: Mostrar mensagem genérica ("Se o email estiver registado, receberá um link...") — não revelar se o email existe ou não (evitar enumeração de utilizadores).
Token expirado: Mostrar mensagem clara "Este link expirou. Por favor, solicite um novo reset de password."
Token já utilizado: Mostrar mensagem "Este link já foi utilizado. Se não foi você, contacte o administrador."
Password não cumpre requisitos: Mostrar erros específicos por cada regra violada.
Passwords não coincidem: Mensagem de erro "As passwords não coincidem."


Requisitos Não Funcionais Recomendados

Segurança:
Usar HTTPS em todo o fluxo
Token deve ser gerado com alta entropia
Implementar proteção contra CSRF no formulário de reset
Rate limiting forte

UX:
Interface responsiva e clara
Contagem regressiva de validade do token na página de reset (opcional mas recomendado)

Auditoria:
Registo completo de todas as tentativas de reset (sucesso e falha)



Queres que eu elabore também a versão técnica mais detalhada (com exemplos de tabelas de base de dados, estrutura do token, endpoints da API, etc.)?
Ou preferes que foque em algum ponto específico (ex: segurança, experiência do utilizador, implementação no backend)?
Diz-me como queres aprofundar!19,3sMarkdownUC-A03: Gestão de Sessão Activa
Users: Admin

Controlo de sessões simultâneas, timeout automático por inactividade e logout manual.

Fluxo principal
1. Utilizador autenticado inicia sessão
→
2. Sistema regista sessão activa
→
3. Após 30min inactividade → aviso
→
4. Após 5min sem resposta → logout automático
→
5. Registo de logout na auditoria

Critérios de aceitação
Timeout de inactividade: 30min (configurável por admin)
Máximo 1 sessão activa por utilizador
Logout forçado pelo Admin disponível
Histórico de sessões visível pelo utilizador

Para utilizadores em campo (tablet/telemóvel), o timeout pode ser ajustado pelo Admin para 60min.Aqui tens uma versão detalhada, clara e profissional do requisito UC-A03: Gestão de Sessão Activa:

UC-A03: Gestão de Sessão Activa
Utilizadores: Administrador (Admin)
Objetivo:
Garantir a segurança das contas através do controlo rigoroso de sessões ativas, timeout automático por inatividade, limitação de sessões simultâneas e registo completo de auditoria.

Fluxo Principal

O utilizador Admin insere as credenciais corretas e autentica-se com sucesso no sistema.
O sistema cria e regista uma nova sessão ativa associada ao utilizador, dispositivo e endereço IP.
O sistema inicia o contador de inatividade (idle timeout).
Após 30 minutos de inatividade, o sistema apresenta um aviso/modal de alerta ao utilizador (ex: “A sua sessão irá terminar em 5 minutos por inatividade”).
Se o utilizador não interagir dentro de 5 minutos após o aviso, o sistema executa o logout automático.
Ao fazer logout (manual ou automático), o sistema:
Invalida a sessão atual
Regista o evento de logout na tabela de auditoria com data/hora, motivo (manual / timeout / forçado) e outros metadados relevantes.

O utilizador é redirecionado para a página de login.


Critérios de Aceitação
1. Timeout de Inatividade

Tempo padrão de inatividade: 30 minutos
Após 30 minutos sem atividade → exibir aviso visível ao utilizador
Após mais 5 minutos sem resposta ao aviso → logout automático
O timeout deve ser configurável a nível global pelo Administrador do sistema
Para utilizadores em campo (tablet ou telemóvel): o Admin pode configurar um timeout específico de 60 minutos

2. Controlo de Sessões Simultâneas

Máximo de 1 sessão ativa por utilizador (política de sessão única)
Ao tentar iniciar sessão num novo dispositivo enquanto já existe uma sessão ativa:
O sistema deve terminar automaticamente a sessão anterior (logout forçado da sessão antiga)
Informar o utilizador no novo login: “A sua sessão anterior foi terminada porque iniciou sessão noutro dispositivo.”

Deve ser possível ao Admin forçar o logout de uma sessão específica de qualquer utilizador

3. Histórico de Sessões

O utilizador Admin deve conseguir visualizar o histórico das suas sessões anteriores, incluindo:
Data e hora de login
Data e hora de logout
Duração da sessão
Tipo de dispositivo / User-Agent
Endereço IP
Motivo do logout (manual, timeout, forçado pelo admin, etc.)

O histórico deve ser acessível numa secção própria (“As Minhas Sessões” ou “Histórico de Sessões”)

4. Logout Forçado pelo Admin

O Administrador do sistema (Super Admin) deve ter a capacidade de:
Visualizar todas as sessões ativas de todos os utilizadores
Forçar o logout de qualquer sessão específica
Forçar o logout de todas as sessões de um determinado utilizador



Requisitos Não Funcionais

Segurança:
Todas as sessões devem ter um session_id único e seguro (recomendado: UUID + assinatura)
Armazenar sessões ativas numa tabela dedicada (active_sessions)
Invalidar sessão no servidor (não apenas no cliente)
Registar todos os eventos de login, logout e timeout em logs de auditoria

Experiência do Utilizador:
Aviso de timeout deve ser claro, não intrusivo e com opção de “Manter sessão ativa”
Em dispositivos móveis, o aviso deve ser adaptado (modal ou toast)
Detectar atividade real do utilizador (movimento do rato, toques no ecrã, scrolls, requests ao servidor, etc.)

Configuração:
Timeout padrão (30 min) e timeout para utilizadores em campo (60 min) devem ser configuráveis via painel de administração
Possibilidade de definir timeout diferente por perfil/grupo de utilizadores no futuro



Fluxos Alternativos / Exceções

Nova sessão iniciada noutro dispositivo:
→ Sessão anterior é terminada automaticamente + notificação enviada por email (recomendado)
Utilizador clica em “Manter sessão ativa” no aviso:
→ Contador de inatividade é reiniciado
Logout manual pelo utilizador:
→ Registo com motivo “Logout manual”
Logout por timeout:
→ Registo com motivo “Timeout de inatividade”
Sessão expirada ao tentar aceder a uma página protegida:
→ Redirecionar para login com mensagem “A sua sessão expirou por inatividade.”


Sugestões de Melhoria (Boas Práticas)

Implementar refresh token + access token (JWT ou session-based com backend)
Usar sliding expiration ou reset do timer a cada request válido
Enviar notificação por email quando uma sessão é terminada forçadamente
Permitir ao utilizador terminar todas as sessões ativas de uma só vez (“Terminar todas as sessões”)
Armazenar o user_agent e ip_address para melhor rastreabilidade



Queres que eu faça também a versão técnica detalhada deste requisito?
Posso incluir:

Estrutura sugerida da tabela active_sessions
Estrutura da tabela de histórico de sessões
Endpoints da API necessários
Regras de invalidação de sessão
Como implementar o contador de inatividade no frontend e backend

Diz-me como queres prosseguir ou se queres ajustar algum ponto (ex: tempos, número de sessões permitidas, notificações, etc.).
Estou à tua disposição!22,9sMarkdownUC-A04: Bloqueio e Desbloqueio de Conta
Users: Admin

Bloqueio automático após tentativas falhadas e desbloqueio manual pelo Administrador.

Fluxo principal
1. 5 tentativas falhadas → bloqueio automático
→
2. Email de notificação ao utilizador
→
3. Utilizador contacta Admin
→
4. Admin desbloqueia com registo do motivo
→
5. Registo na auditoria

Critérios de aceitação
Bloqueio automático: 5 tentativas falhadas
Desbloqueio apenas por Admin
Motivo de desbloqueio obrigatório
Notificação ao utilizador após desbloqueio

Admin não pode desbloquear a própria conta — exige um segundo Admin.Aqui tens uma versão detalhada, clara e profissional do requisito UC-A04: Bloqueio e Desbloqueio de Conta:

UC-A04: Bloqueio e Desbloqueio de Conta
Utilizadores: Administrador (Admin)
Objetivo:
Proteger as contas contra ataques de força bruta através de bloqueio automático após tentativas falhadas de login e permitir o desbloqueio controlado exclusivamente por um Administrador, com registo completo de auditoria.

Fluxo Principal
  1. O utilizador (Admin) tenta efetuar login na plataforma.
  2. Após 5 tentativas consecutivas falhadas de login com a mesma conta, o sistema bloqueia automaticamente a conta.
  3. O sistema envia imediatamente um email de notificação ao utilizador informando que a sua conta foi bloqueada por tentativas excessivas de login.
  4. O utilizador contacta o Administrador para solicitar o desbloqueio da conta.
  5. O Administrador acede à secção de gestão de utilizadores, localiza a conta bloqueada e executa a ação de desbloqueio.
  6. O sistema exige que o Admin preencha obrigatoriamente o motivo do desbloqueio.
  7. O sistema desbloqueia a conta, regista o evento na auditoria e envia uma notificação por email ao utilizador informando que a conta foi desbloqueada com sucesso.
  8. O utilizador pode agora voltar a efetuar login normalmente.

Critérios de Aceitação
1. Bloqueio Automático
  1.1 A conta é bloqueada automaticamente após 5 tentativas falhadas consecutivas de login.
  1.2 O contador de tentativas falhadas deve ser reiniciado após um login bem-sucedido ou após um período de tempo configurável (ex: 30 minutos sem tentativas).
  1.3 O bloqueio é temporalmente indefinido (só pode ser removido manualmente por um Admin).
  1.4 Durante o bloqueio, qualquer tentativa de login deve ser rejeitada com a mensagem:
    “Esta conta encontra-se bloqueada. Contacte o administrador do sistema.”
2. Desbloqueio de Conta
  2.1 O desbloqueio de contas só pode ser realizado por um Administrador.
  2.2 Motivo do desbloqueio é obrigatório (campo de texto livre com mínimo de caracteres).
  2.3 Após o desbloqueio, o sistema deve:
    2.3.1 Registar o evento na tabela de auditoria (quem desbloqueou, quando, motivo, IP, etc.)
    2.3.2 Enviar notificação por email ao utilizador informando que a conta foi desbloqueada.
  2.4 O Admin não pode desbloquear a sua própria conta. → Exige que outro Admin (segundo administrador) realize o desbloqueio.
3. Notificações
  3.1 Email de bloqueio: Enviado automaticamente ao utilizador quando a conta é bloqueada.
  3.2 Email de desbloqueio: Enviado ao utilizador após o desbloqueio bem-sucedido, incluindo data/hora e nome do Admin que desbloqueou.

  