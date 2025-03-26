---
title: "Cross-site request forgery (CSRF)"
---

# Cross-site request forgery (CSRF)

## Table of contents

-   [Introdução](#introdução)
    -   [Cross-site vs cross-origin](#cross-site-vs-cross-origin)
-   [Prevenção](#prevenção)
    -   [Tokens Anti-CSRF](#tokens-anti-csrf)
    -   [Cookies de envio duplo assinados](#cookies-de-envio-duplo-assinados)
    -   [Cabeçalho Origin](#cabeçalho-origin)
-   [Atributo SameSite em cookies](#atributo-same-site-em-cookies)

## Introdução

Ataques CSRF (_Cross-Site Request Forgery_), que em português significa "Falsificação de Solicitação entre Sites", permitem que um invasor faça solicitações autenticadas em nome de usuários quando suas credenciais são armazenadas em cookies.

Quando um _client_ faz uma solicitação entre origens, o navegador envia uma solicitação de pré-voo para verificar se a solicitação é permitida (CORS). No entanto, para certas solicitações "simples", incluindo envios de formulário, essa etapa é omitida. Como _cookies_ são incluídos automaticamente mesmo para solicitações entre origens, isso permite que um ator malicioso faça solicitações como se fosse o usuário autenticado mesmo sem nunca roubar diretamente o token de qualquer domínio. A [política de mesma origem](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) proíbe _clients_ entre origens de ler respostas por padrão, mas a solicitação ainda é processada.

Por exemplo, se você estiver conectado ao `banco.com`, seu cookie de sessão será enviado junto com o envio deste formulário, mesmo que o formulário esteja hospedado em um domínio diferente.

```html
<form action="https://banco.com/enviar-dinheiro" method="post">
	<input name="receptor" value="hacker_do_mal" />
	<input name="valor" value="R$100" />
	<button>Enviar dinheiro</button>
</form>
```

Também pode ser apenas uma solicitação `fetch()` para que nenhuma entrada do usuário seja necessária.

```ts
const body = new URLSearchParams();
body.set("receptor", "hacker_do_mal");
body.set("valor", "R$100");

await fetch("https://banco.com/enviar-dinheiro", {
	method: "POST",
	body,
});
```

### Cross-site vs cross-origin

Enquanto requisições entre 2 domínios totalmente diferentes são consideradas tanto _cross-site_ quanto _cross-origin_, aquelas entre 2 subdomínios não são consideradas _cross-site_, mas são consideradas requisições _cross-origin_. Embora o nome "falsificação de solicitação entre sites" implique requisições _cross-site_, você deve ser estrito por padrão e proteger sua aplicação contra ataques _cross-origin_ também.

## Prevenção

CSRF podem ser previnidos aceitando apenas requisições POST e similares feitas por navegadores de uma origem confiável.

A proteção deve ser implementada para todas as rotas que lidam com formulários. Se sua aplicação não usa formulários atualmente, ainda pode ser uma boa ideia pelo menos [verificar o cabeçalho `Origin`](#cabeçalho-origin) para prevenir problemas futuros. Também é uma boa ideia, em geral, modificar recursos apenas usando métodos POST e similares (PUT, DELETE, etc).

Para a abordagem com _tokens_, o _token_ não deve ser de uso único (por exemplo, um novo _token_ para cada envio de formulário) pois quebrará com um único clique no botão de voltar. É crucial que suas páginas tenham uma política estrita de compartilhamento de recursos entre origens (CORS). Se `Access-Control-Allow-Credentials` não for estrito, um site malicioso pode enviar uma solicitação GET para obter um formulário HTML com um _token_ CSRF válido.

### Tokens Anti-CSRF

Esse é um método muito simples onde cada sessão tem um [token](/server-side-tokens) CSRF único associado a ela.

```html
<form method="post">
	<input name="mensagem" />
	<input type="hidden" name="__csrf" value="<TOKEN_CSRF>" />
	<button>Enviar</button>
</form>
```

### Cookies de envio duplo assinados

Se armazenar o _token_ no servidor não for uma opção, usar cookies de envio duplo assinados é outra abordagem. Isso é diferente do cookie de envio duplo básico no qual o _token_ incluído no formulário é assinado com um segredo.

Um novo [_token_](/server-side-tokens) é gerado e transformado em um _hash_ com HMAC SHA-256 usando uma chave secreta. Cada HMAC deve estar vinculado à sessão do usuário. Você também pode criptografar o _token_ com algoritmos como AES.

```go
func generateCSRFToken(sessionId string) (string, []byte) {
	buffer := [10]byte{}
	crypto.rand.Read(buffer)
	csrfToken := base64.StdEncoding.encodeToString(buffer)
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(csrfToken + "." + sessionId))
	csrfTokenHMAC := mac.Sum(nil)
	return csrfToken, csrfTokenHMAC
}
```

The token is stored as a cookie and the HMAC is embedded in the form. The cookie should have the `Secure`, `HttpOnly`, and `SameSite` attribute. To validate a request, the cookie can be used to verify the signature sent in the form data.

Esse _token_ é armazenado como um cookie e o HMAC é incorporado no formulário. O cookie deve ter os atributos `Secure`, `HttpOnly` e `SameSite`. Para validar uma requisição, o cookie pode ser usado para verificar a assinatura enviada nos dados do formulário.

#### Cookies de envio duplo tradicionais

Cookies de envio duplo tradicionais que não são assinados ainda deixarão você vulnerável se um atacante tiver acesso a um subdomínio do domínio de sua aplicação. Isso permitiria que eles definissem seus próprios cookies de envio duplo.

### Cabeçalho Origin

Um jeito muito simples de se prevenir contra ataques CSRF é checar o cabeçalho `Origin` das requisições que não são GET. Esse é um cabeçalho relativamente novo que inclui a [origem](https://developer.mozilla.org/en-US/docs/Glossary/Origin) da requisição. Se você depender desse cabeçalho, é crucial que sua aplicação não use requisições GET para modificar recursos.

Enquanto o cabeçalho `Origin` pode ser falsificado usando um cliente personalizado, a parte importante é que isso não pode ser feito usando JavaScript do lado do cliente. Usuários só são vulneráveis a CSRF quando usam um navegador.

```go
func handleRequest(w http.ResponseWriter, request *http.Request) {
  	if request.Method != "GET" {
		originHeader := request.Header.Get()
		// Você também pode compará-lo com o Host ou o cabeçalho X-Forwarded-Host.
		if originHeader != "https://example.com" {
			// Origem não confiável
			w.WriteHeader(403)
			return
		}
  	}
  	// ...
}
```

O cabeçalho `Origin` tem sido suportado por todos os navegadores modernos desde por volta de 2020, embora Chrome e Safari o tenham suportado antes disso. Se o cabeçalho `Origin` não estiver incluído, não permita a requisição.

O cabeçalho `Referer` é um cabeçalho similar introduzido antes do cabeçalho `Origin`. Isso pode ser usado como um _fallback_ se o cabeçalho `Origin` não estiver definido.

## Atributo SameSite em cookies

_Cookies_ de sessão devem ter o atributo `SameSite`. Esse atributo determina quando o navegador inclui o cookie nas requisições. _Cookies_ `SameSite=Lax` só serão enviados em requisições entre sites se a requisição usar um [método HTTP seguro](https://developer.mozilla.org/en-US/docs/Glossary/Safe/HTTP) (como GET), enquanto _cookies_ `SameSite=Strict` não serão enviados em nenhuma requisição entre sites. Recomendamos usar `Lax` como padrão, já que _cookies_ `Strict` não serão enviados quando um usuário acessar seu site via um link externo.

Se você definir o valor como `Lax`, é crucial que sua aplicação não use requisições GET para modificar recursos. O suporte do navegador para a _flag_ `SameSite` mostra que ela está atualmente disponível para 96% dos usuários da _web_. É importante notar que a bandeira só protege contra falsificação de solicitação entre sites (não falsificação de solicitação entre origens) e geralmente não deve ser sua única camada de defesa.
