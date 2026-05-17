UC-A05: Registo de Auditoria de Acesso
Utilizadores: Sistema (Automático) + Administrador (para consulta e exportação)
Objetivo:
Garantir o registo completo, seguro e imutável de todos os eventos relacionados com autenticação e ações críticas do sistema, de forma a cumprir os requisitos de auditoria e supervisão do Banco Mundial.

Fluxo Principal
  1. Qualquer evento de autenticação ou ação crítica ocorre no sistema (login, logout, falha de login, 2FA, bloqueio de conta, desbloqueio, reset de password, etc.).
  2. O sistema captura automaticamente todos os dados relevantes do evento.
  3. O sistema regista o evento na trilha de auditoria de forma imutável.
  4. O registo inclui pelo menos: utilizador, ação, resultado, IP, User-Agent, timestamp e outros metadados relevantes.
  5. Todos os registos são armazenados de forma segura e protegida contra alteração ou eliminação.
  6. O Administrador pode consultar os logs através de uma interface de pesquisa avançada.
  7. O Administrador pode exportar os logs em formato CSV ou PDF(com Puppeteer) quando necessário.

Critérios de Aceitação
  Eventos a Registar (Obrigatórios)
  1. Devem ser registados todos os seguintes eventos:
    1.1 Autenticação:
      1.1.1 Tentativa de login bem-sucedida
      1.1.2 Tentativa de login falhada (password incorreta)
      1.1.3 Tentativa de 2FA bem-sucedida
      1.1.4 Tentativa de 2FA falhada
      1.1.5 Logout manual
      1.1.6 Logout automático por timeout
      1.1.7 Logout forçado por outra sessão
    1.2. Gestão de Conta:
      1.1.1 Bloqueio automático de conta (por tentativas falhadas)
      1.1.2 Desbloqueio manual de conta
      1.1.3 Reset de password (via email)
      1.1.4 Alteração de password bem-sucedida
    1.3 Ações Críticas:
      1.3.1 Tentativa de desbloqueio da própria conta (proibida)
      1.3.2 Reutilização de token de reset de password
      1.3.3 Qualquer alteração de configurações de segurança

Informações Obrigatórias em Cada Registo
Cada entrada no log de auditoria deve conter no mínimo:
  1. timestamp (data e hora UTC + fuso horário)
  2. user_id e username / email
  3. action (ex: LOGIN_SUCCESS, LOGIN_FAILED, LOGOUT, 2FA_FAILED, etc.)
  4. result (SUCCESS / FAILED)
  5. ip_address
  6. user_agent
  7. session_id (quando aplicável)
  8. details (campo JSON ou texto com informações adicionais)
  9. device_info (opcional, mas recomendado)
  10. location (cidade/país estimado pelo IP – opcional)

Imutabilidade e Segurança
  1. Os logs devem ser imutáveis: uma vez registados, não podem ser editados nem eliminados.
  2. Implementar mecanismo de proteção contra alteração (ex: hash chain, append-only, ou Write-Once-Read-Many - WORM).
  3. Retenção mínima dos logs: 5 anos (requisito do Banco Mundial).
  4. Proteção contra exclusão acidental ou maliciosa (mesmo por Super Admin).

Consulta e Exportação
  1. Interface de consulta com filtros avançados (data, utilizador, ação, resultado, IP, etc.).
  2. Possibilidade de exportar os resultados em CSV e PDF(com Puppeteer).
  3. Os logs devem estar disponíveis para exportação em menos de 24 horas após solicitação do Banco Mundial.


Requisitos Não Funcionais
  1. Imutabilidade: Recomenda-se o uso de:
    1.1 Tabela append-only na base de dados
    1.2 Armazenamento em sistemas como WORM, Blockchain lite, ou append-only logs com hash chaining
    1.3 Backup diário em storage imutável (ex: AWS S3 Object Lock, Azure Blob Immutable)
  2. Desempenho: O registo de auditoria não deve impactar significativamente o tempo de resposta das operações.
  3. Conformidade: Atender aos requisitos de auditoria e supervisão do Banco Mundial.
  4. Segurança:
    4.1 Acesso aos logs restrito apenas a Administradores com permissão específica.
    4.2 Todas as consultas aos logs devem também ser auditadas.

Item,Detalhe,Obrigatório
Imutabilidade dos logs,Não permite edição ou eliminação,Sim
Retenção mínima,5 anos,Sim
Eventos de autenticação,"Todos (login, logout, falhas, 2FA, etc.)",Sim
Informações por registo,"IP, User-Agent, timestamp, resultado, utilizador, ação",Sim
Exportação,CSV e PDF(com Puppeteer),Sim
Disponibilidade para BM,Logs disponíveis em < 24 horas após solicitação,Sim
Registo de ações críticas,"Reset password, bloqueio/desbloqueio, etc.",Sim


Recomendações Adicionais
  1. Criar categorias de criticidade dos eventos (Baixa, Média, Alta, Crítica).
  2. Implementar alertas em tempo real para eventos suspeitos (ex: múltiplos logins falhados de IPs diferentes, tentativas de reutilização de token, etc.).
  3. Possibilitar pesquisa full-text nos logs.
  4. Manter um dashboard de “Últimos Acessos” e “Atividade Recente” para os Admins.