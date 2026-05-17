UC-A02: Recuperação de Password
Utilizadores: Todos
Objetivo: Permitir que o utilizador recupere o acesso à sua conta de forma segura através de um processo de reset de password via email, utilizando um token temporário de uso único.

Fluxo Principal (Happy Path)

  1. O utilizador acede à página de login e clica no link "Esqueci a password".
  2. O sistema solicita o email associado à conta.
  3. O sistema valida se o email existe na base de dados.
  4. O sistema gera um token de reset:
    4.1. Único (UUID v4 ou equivalente criptograficamente seguro)
    4.2. Válido por 15 minutos
    4.3. Armazenado na base de dados associado ao utilizador (com data de expiração e flag used = false)
  5. O sistema envia um email para o utilizador contendo:
    5.1 Um link de reset seguro (ex: https://app.dominio.com/reset-password?token=abc123...)
    5.2 Tempo de validade do link (15 minutos)
    5.3 Aviso para não partilhar o link
  6. O utilizador clica no link recebido no email.
  7. O sistema valida o token:
    7.1 Verifica se existe, não está expirado e ainda não foi utilizado.
  8. Se válido, o sistema apresenta o formulário para definir nova password.
  9. O utilizador introduz a nova password e confirma-a.
  10. O sistema valida os requisitos de complexidade da password.
  11. O sistema atualiza a password do utilizador (com hash seguro — recomendado: Argon2id ou bcrypt).
  12. O sistema invalida imediatamente o token (used = true ou elimina o registo).
  13. O sistema envia uma notificação de confirmação por email ao utilizador informando que a password foi alterada com sucesso.
  14. O utilizador é redirecionado para a página de login.

Critérios de Aceitação (Acceptance Criteria)
  1. Segurança do Token
    1.1 O token deve ter validade máxima de 15 minutos a partir do momento da sua geração.
    1.2 O token é de uso único: após o primeiro acesso bem-sucedido ao link, deve ser imediatamente invalidado.
    1.3 Tentativa de reutilização do mesmo token (mesmo que ainda dentro do prazo de 15min) deve ser rejeitada.
    1.4 Qualquer tentativa de reutilização de token inválido/expirado/usado deve registar um alerta de segurança no sistema (log de segurança + eventual notificação ao admin ou equipa de segurança).
  2. Requisitos da Nova Password
    2.1 A nova password deve cumprir os seguintes critérios mínimos:
      2.1.1 Mínimo de 8 caracteres
      2.1.2 Pelo menos 1 letra maiúscula (A-Z)
      2.1.3 Pelo menos 1 número (0-9)
      2.1.4 Pelo menos 1 caractere especial (! @ # $ % ^ & * ( ) - _ = + [ ] { } | ; : ' " , . < > / ?)
  3. Outros Critérios
    3.1 O link de reset deve conter o token como parâmetro de query string de forma segura.
    3.2 O sistema deve impedir ataques de rate limiting na funcionalidade de "Esqueci a password" (ex: máximo 3 tentativas por email por hora).
    3.3 Todas as operações sensíveis (geração de token, validação, alteração de password) devem ser devidamente registadas em logs de auditoria.
    3.4 A password deve ser armazenada sempre com hash + salt (nunca em texto plano).
    3.5 Após o reset bem-sucedido, o utilizador deve receber um email de confirmação com informações como:
      3.5.1 Data e hora da alteração
      3.5.2 Recomendação para alterar a password regularmente
      3.5.3 Alerta caso não tenha sido ele a efetuar o reset (para detetar possíveis compromissos)

Fluxos Alternativos / Exceções
  1. Email não encontrado: Mostrar mensagem genérica ("Se o email estiver registado, receberá um link...") — não revelar se o email existe ou não (evitar enumeração de utilizadores).
  2. Token expirado: Mostrar mensagem clara "Este link expirou. Por favor, solicite um novo reset de password."
  3. Token já utilizado: Mostrar mensagem "Este link já foi utilizado. Se não foi você, contacte o administrador."
  4. Password não cumpre requisitos: Mostrar erros específicos por cada regra violada.
  5. Passwords não coincidem: Mensagem de erro "As passwords não coincidem."

Requisitos Não Funcionais Recomendados
  1. Segurança:
    1.1 Usar HTTPS em todo o fluxo
    1.2 Token deve ser gerado com alta entropia
    1.3 Implementar proteção contra CSRF no formulário de reset
    1.4 Rate limiting forte
  2. UX:
    2.1 Interface responsiva e clara
    2.2 Contagem regressiva de validade do token na página de reset (opcional mas recomendado)
  3. Auditoria:
    3.1 Registo completo de todas as tentativas de reset (sucesso e falha)
    