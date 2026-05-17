UC-A03: Gestão de Sessão Activa
Utilizadores: Todos
Objetivo:
Garantir a segurança das contas através do controlo rigoroso de sessões ativas, timeout automático por inatividade, limitação de sessões simultâneas e registo completo de auditoria.


Fluxo Principal
  1. O utilizador insere as credenciais corretas e autentica-se com sucesso no sistema.
  2. O sistema cria e regista uma nova sessão ativa associada ao utilizador, dispositivo e endereço IP.
  3. O sistema inicia o contador de inatividade (idle timeout).
  4. Após 30 minutos de inatividade, o sistema apresenta um aviso/modal de alerta ao utilizador (ex: “A sua sessão irá terminar em 5 minutos por inatividade”).
  5. Se o utilizador não interagir dentro de 5 minutos após o aviso, o sistema executa o logout automático.
  6. Ao fazer logout (manual ou automático), o sistema:
    6.1 Invalida a sessão atual
    6.2 Regista o evento de logout na tabela de auditoria com data/hora, motivo (manual / timeout / forçado) e outros metadados relevantes.
  7. O utilizador é redirecionado para a página de login.


Critérios de Aceitação
  1. Timeout de Inatividade
    1.1 Tempo padrão de inatividade: 30 minutos
    1.2 Após 30 minutos sem atividade → exibir aviso visível ao utilizador
    1.3 Após mais 5 minutos sem resposta ao aviso → logout automático
    1.4 O timeout deve ser configurável a nível global pelo Administrador do sistema
    1.5 Para utilizadores em campo (tablet ou telemóvel): o Admin pode configurar um timeout específico de 60 minutos
  2. Controlo de Sessões Simultâneas
    2.1 Máximo de 1 sessão ativa por utilizador (política de sessão única)
    2.2 Ao tentar iniciar sessão num novo dispositivo enquanto já existe uma sessão ativa:
      2.1.1 O sistema deve terminar automaticamente a sessão anterior (logout forçado da sessão antiga)
      2.1.2 Informar o utilizador no novo login: “A sua sessão anterior foi terminada porque iniciou sessão noutro dispositivo.”
    2.3 Deve ser possível ao Admin forçar o logout de uma sessão específica de qualquer utilizador
  3. Histórico de Sessões
    3.1 O utilizador Admin deve conseguir visualizar o histórico das suas sessões anteriores, incluindo:
      3.1.1 Data e hora de login
      3.1.2 Data e hora de logout
      3.1.3 Duração da sessão
      3.1.4 Tipo de dispositivo / User-Agent
      3.1.5 Endereço IP
      3.1.6 Motivo do logout (manual, timeout, forçado pelo admin, etc.)
    3.2 O histórico deve ser acessível numa secção própria (“As Minhas Sessões” ou “Histórico de Sessões”)
  4. Logout Forçado pelo Admin
    4.1 O Administrador do sistema (Super Admin) deve ter a capacidade de:
      4.1.1 Visualizar todas as sessões ativas de todos os utilizadores
      4.1.2 Forçar o logout de qualquer sessão específica
      4.1.3 Forçar o logout de todas as sessões de um determinado utilizador


Requisitos Não Funcionais
  1. Segurança:
    1.1 Todas as sessões devem ter um session_id único e seguro (recomendado: UUID + assinatura)
    1.2 Armazenar sessões ativas numa tabela dedicada (active_sessions)
    1.3 Invalidar sessão no servidor (não apenas no cliente)
    1.4 Registar todos os eventos de login, logout e timeout em logs de auditoria
  2. Experiência do Utilizador:
    2.1 Aviso de timeout deve ser claro, não intrusivo e com opção de “Manter sessão ativa”
    2.2 Em dispositivos móveis, o aviso deve ser adaptado (modal ou toast)
    2.3 Detectar atividade real do utilizador (movimento do rato, toques no ecrã, scrolls, requests ao servidor, etc.)
  3. Configuração:
    3.1 Timeout padrão (30 min) e timeout para utilizadores em campo (60 min) devem ser configuráveis via painel de administração
    3.2 Possibilidade de definir timeout diferente por perfil/grupo de utilizadores no futuro