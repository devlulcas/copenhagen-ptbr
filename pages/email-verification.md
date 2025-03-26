---
title: "Verificação de e-mail"
---

# Verificação de e-mail

Se sua aplicação requer que endereços de e-mail de usuários sejam únicos, a verificação de e-mail é essencial. Ela desencoraja usuários de inserir um endereço de e-mail aleatório e, se a redefinição de senha for implementada, permite que usuários recuperem contas criadas com seu endereço de e-mail. Você pode até querer bloquear usuários de acessar o conteúdo de sua aplicação até que verifiquem seu endereço de e-mail.

_Endereços de e-mail são insensíveis a maiúsculas e minúsculas._ Recomendamos normalizar os endereços de e-mail fornecidos pelo usuário para minúsculas.

## Tabela de conteúdos

-   [Validação de entradas](#validação-de-entradas)
    -   [Sub-endereçamento](#sub-endereçamento)
-   [Códigos de verificação de e-mail](#códigos-de-verificação-de-e-mail)
-   [Links de verificação de e-mail](#links-de-verificação-de-e-mail)
-   [Alteração de e-mails](#alteração-de-e-mails)
-   [Limitação de taxa](#limitação-de-taxa)

## Validação de Entradas

E-mails são complexos e não podem ser totalmente validados usando Regex. Tentar usar Regex pode também introduzir [vulnerabilidades ReDoS](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS). Não complique as coisas:

-   Inclui pelo menos 1 caractere `@`.
-   Tem pelo menos 1 caractere antes do `@`.
-   A parte do domínio inclui pelo menos 1 `.` e tem pelo menos 1 caractere antes dele.
-   Não começa ou termina com um espaço em branco.
-   Máximo de 255 caracteres.

### Sub-endereçamento

Alguns provedores de e-mail, incluindo o Google, permitem que os usuários especifiquem uma tag que será ignorada por seus servidores. Por exemplo, um usuário com `user@example.com` pode usar `user+foo@example.com` e `user+bar@example.com`. Você pode bloquear e-mails com `+` para impedir que os usuários criem múltiplas contas com o mesmo endereço de e-mail, mas os usuários ainda poderiam usar endereços de e-mail temporários ou simplesmente criar um novo endereço de e-mail. Nunca remova silenciosamente a parte da tag da entrada do usuário, pois um endereço de e-mail com `+` pode ser um endereço de e-mail regular e válido.

## Códigos de Verificação de E-mail

Uma maneira de verificar o e-mail é enviar um código secreto armazenado no servidor para a caixa de correio do usuário.

Esta abordagem tem algumas vantagens em comparação com o uso de links:

-   As pessoas estão cada vez menos propensas a clicar em links.
-   Alguns filtros podem classificar automaticamente e-mails com links como spam ou phishing.
-   O uso de links de verificação pode introduzir atrito se o usuário quiser concluir o processo em um dispositivo que não tenha acesso à mensagem de verificação, ou em um dispositivo que não possa abrir links.

O código de verificação deve ter pelo menos 8 dígitos se for numérico, e pelo menos 6 dígitos se for alfanumérico. Use um código mais forte se a verificação fizer parte de um processo seguro, como criar uma nova conta ou alterar informações de contato. Você deve evitar usar letras minúsculas e maiúsculas. Você também pode querer remover números e letras que possam ser mal interpretados (0, O, 1, I, etc.). O código deve ser gerado usando um gerador aleatório criptograficamente seguro.

Um único código de verificação deve estar vinculado a um único usuário e e-mail. Isso é especialmente importante se você permitir que os usuários alterem seu endereço de e-mail após o envio de um e-mail. Cada código deve ser válido por pelo menos 15 minutos (recomenda-se entre 1-24 horas). O código deve ser de uso único e imediatamente invalidado após a validação. Um novo código de verificação deve ser gerado cada vez que o usuário solicitar outro e-mail/código.

Semelhante a um formulário de login regular, deve ser implementada uma limitação ou restrição de taxa com base no ID do usuário. Um bom limite é em torno de 10 tentativas por hora. Assumindo que a limitação adequada seja implementada, o código pode ser válido por até 24 horas. Você deve gerar e reenviar um novo código se o código fornecido pelo usuário tiver expirado.

Todas as sessões de um usuário devem ser invalidadas quando seu e-mail for verificado.

## Links de Verificação de E-mail

Uma forma alternativa de verificar e-mails é usar um link de verificação que contém um [_token_](/server-side-tokens) longo, aleatório e de uso único.

```
https://example.com/verify-email/<TOKEN>
```

Um único token deve estar vinculado a um único usuário e e-mail. Isso é especialmente importante se você permitir que os usuários alterem seu endereço de e-mail após o envio de um e-mail. Os tokens devem ser de uso único e ser imediatamente excluídos do armazenamento após a verificação. O token deve ser válido por pelo menos 15 minutos (recomenda-se entre 1-24 horas). Quando um usuário solicitar outro e-mail de verificação, você pode reenviar o token anterior em vez de gerar um novo token se esse token ainda estiver dentro do prazo de validade.

Certifique-se de definir a tag de [Política de Referência](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy) como `strict-origin` (ou equivalente) para qualquer caminho que inclua tokens para proteger os tokens de vazamento de referência (do inglês: _*referer leakage*_).

Todas as sessões devem ser invalidadas quando o e-mail for verificado (e criar uma nova para o usuário atual para que continuem conectados).

## Alteração de E-mails

O usuário deve ser solicitado a fornecer sua senha, ou se a [autenticação de múltiplos fatores](/mfa) estiver habilitada, autenticado com um de seus fatores secundários. O novo e-mail deve ser armazenado separadamente do e-mail atual até que seja verificado. Por exemplo, o novo e-mail pode ser armazenado com o token/código de verificação.

Uma notificação deve ser enviada para o endereço de e-mail anterior quando o usuário alterar seu e-mail.

## Limitação de Taxa

Qualquer _endpoint_ que possa enviar e-mails deve ter uma limitação de taxa estrita implementada.
