UC-A01: Autenticação com 2FA
Utilizadores: Todos
Objetivo:
Garantir um nível elevado de segurança no acesso à plataforma através de autenticação em dois fatores (2FA), tornando-o obrigatório para todos os utilizadores, conforme exigência do Banco Mundial.

Administrador ativar a obrigacao do login em 2 factores, se estiver falso deixar o login directo com email e senha

Fluxo Principal
  1. O utilizador acede à página de login e insere o email e a password.
  2. O sistema valida as credenciais (email + password).
  3. Se as credenciais estiverem corretas, o sistema verifica se o 2FA está ativado (obrigatório para todos).
  4. O sistema solicita a introdução do código OTP (One-Time Password).
  5. O utilizador insere o código de 6 dígitos gerado pela aplicação autenticadora (Google Authenticator, Microsoft Authenticator, Authy, etc.) ou recebido via SMS.
  6. O sistema valida o código OTP.
  7. Se o código estiver correto, o sistema emite um Token JWT com duração de 8 horas.
  8. O utilizador é autenticado e redirecionado para o dashboard da aplicação.
  9. O sistema regista o login completo na trilha de auditoria.

Critérios de Aceitação
1. Obrigatoriedade do 2FA
  1.1 A autenticação com 2FA é obrigatória para todos os perfis.
  1.2 Não é possível desativar o 2FA.
  1.3 No primeiro login após a ativação do 2FA, o utilizador deve configurar o autenticador (onboarding do 2FA).
2. Métodos de 2FA Suportados
  2.1 Aplicação Autenticadora (TOTP) – Preferencial (Google Authenticator, Microsoft Authenticator, etc.)
  2.2 SMS – Método alternativo (para utilizadores que não consigam usar app autenticadora)
3. Gestão de Tentativas Falhadas
  3.1 Máximo de 5 tentativas falhadas do código OTP.
  3.2 Após 5 tentativas incorretas, a conta deve ser bloqueada temporariamente por 30 minutos.
  3.3 O bloqueio temporário aplica-se apenas ao 2FA (o utilizador ainda pode tentar login com email/password após os 30 minutos).
4. Token JWT
  4.1 Expiração do Access Token: 8 horas (480 minutos)
  4.2 O token deve ser renovável (refresh token mechanism recomendado)
  4.3 O JWT deve conter claims mínimas: user_id, role, session_id, iat, exp
  4.4 Implementar invalidação ativa de tokens (token revocation list) quando necessário
5. Auditoria
  5.1 Registo completo de cada tentativa de login na trilha de auditoria, incluindo:
    5.1.1 Sucesso ou falha
    5.1.2 Tipo de falha (password incorreta ou OTP incorreto)
    5.1.3 Endereço IP
    5.1.4 User-Agent / Dispositivo
    5.1.5 Timestamp
    5.1.6 Método 2FA utilizado (TOTP ou SMS)
    