# Servidor Springs de Autorização
Um servidor para autorização usando Oauth2, e utilizando Gradle. O objetivo é um, criar um token de acesso ao usuário.

Para obter o JWS (JSON Web Token) vamos utilizar o fluxo **authorization code grant**. Neste fluxo o usuário vai ser redirecionado para uma outra página onde ele irá logar e escolher os acessos que permitirá. Com a autorização do usuário, o servidor irá redirecionar de volta para a página da aplicação original, com uma header de ***"Authorization"***.

## Dependencia
```
dependencies {
     implementation 'org.springframework.boot:spring-boot-starter-oauth2-authorization-server'
}
```

## Alteração de Porta
Para evitar conflitos lindos, alterei a porta para 9000.
```
server:
   port: 9000
```

## Configuração
Toda configuração foi colocado na classe ***AuthorizationServerConfig***.

### Usuários Pré Definidos
Para simplificar cirie uns usuários, mas o ideial seria ir buscá-los em um banco de dados, LDAP, etc.
```
@Bean
	public UserDetailsService userDetailsService(PasswordEncoder encoder) {
		UserDetails user1 = User.builder().username("rod").password(encoder.encode("pass")).roles("USER").build();

		UserDetails user2 = User.builder().username("dan").password(encoder.encode("pass")).roles("USER").build();

		return new InMemoryUserDetailsManager(user1, user2);
	}
```

### Padrões do Servidor de Autorização
Utilizando padrões básicos e foi utilizado HIGHEST_PRECEDENCE para garantir que esta será utilizada (no caso de existir duplicações).
```
@Bean
	@Order(Ordered.HIGHEST_PRECEDENCE)
	public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
		OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

		http.getConfigurer(OAuth2AuthorizationServerConfigurer.class).oidc(Customizer.withDefaults());

		return http.formLogin(form -> Customizer.withDefaults()).build();
	}
```
### Implementação na Memória
Foi utilizado implementação básica (na memória), mas o ideal é ir em um banco de dados ou algo do tipo.
```
@Bean
	public RegisteredClientRepository registeredClientRepository(PasswordEncoder encoder) {
		RegisteredClient rc = RegisteredClient.withId(UUID.randomUUID().toString())
				.clientId("rod-admin-client")
				.clientSecret(encoder.encode("pass"))
				.clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
				.redirectUri("http://localhost:8080/login/oauth2/admin-client")
				.scope("write")
				.scope("delete")
				.scope(OidcScopes.OPENID)
				.clientSettings(ClientSettings.builder().requireAuthorizationConsent(true).build()).build();

		return new InMemoryRegisteredClientRepository(rc);
	}
```
O que ocorre ai arriba é:
- criação de um identificador randômico
- definição de um nome para nosso cliente
- senha do cliente
- definição e autenticação básica (usuário/senha)
- habilitando tanto codigo como refresh token
- redirecionamento de URL habilitado
- quais recursos o usuário vai poder permitir

## teste
Acesse a URL abaixo (ou curl)
http://localhost:9000/oauth2/authorize?response_type=code&client_id=rod-admin-client&redirect_uri=http://localhost:8080/login/oauth2/admin-client&scope=delete+write

Após logar, vc vai ver um erro. Normal pois não temos um client. Mas a parte importante aqui é pgar o token, que você vai conseguir.
> curl http://localhost:9000/oauth2/token 
-H "Content-type: application/x-www-form-urlencoded"
-d "grant_type=authorization_code" 
-d "redirect_uri=http://localhost:8080/login/oauth2/admin-client"
-d "code=uEaSvAXma9kaf7ONYCE8gvmHVG751fmLiKR-YbryZNz0HVZrK9uY5oI7Pm3rK82JtWbX9epMQpGSMs5o4cLQEoIanr7MLAnV-KHPw-RJ3nLmhaqnTGaYFRPM3qvzInWQ"
-u rod-admin-client:pass2

olha a resposta (devtools se estiver no browser)
>{
  "access_token": "eyJraWQiOiIwYmRkM ...",
  "refresh_token": "vFm86h9e7mbhyhGCO ...",
  "scope": "delete write",
  "token_type": "Bearer",
  "expires_in": 300
}

Para ver o que tem no token do JWT, vai em https://jwt.io

### Pegando outro token
Se quiser um outro token de acesso, utiliza o refresh token.
> $ curl localhost:9000/oauth2/token \
-H"Content-type: application/x-www-form-urlencoded" \
-d"grant_type=refresh_token&refresh_token=10Hqw34..." \
-u rod-admin-client:pass2

## Definindo o timeout do token
Abaixo
```
@Bean
public RegisteredClientRepository registeredClientRepository(PasswordEncoder encoder) {
   return <bla-bla-bla>.tokenSettings(TokenSettings.builder().accessTokenTimeToLive(Duration.ofMinutes(45L)).build())
}
```

## Habilitando SegurançaEnabling Securing API with Resource Server
Primeiro adicione esta dependência
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

Segundo passo.
```
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
	return http
		.authorizeHttpRequests(
				auth -> auth.requestMatchers(HttpMethod.GET, "/restapi/test").hasAuthority("SCOPE_write"))
		.oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults())).build();
}
```
Não confie em qualquer token, então especifique no cliente qual a URL de confiança.
```
spring:
   security:
      oauth2:
         resourceserver:
            jwt:
               jwk-set-uri: http:/ /localhost:9000/oauth2/jwks
```

Findo. Noix.