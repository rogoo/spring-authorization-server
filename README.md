# spring-authorization-server
An Authorization Server using Oauth2 to authorize access to resources, build using Gradle.

An Authorization Server has one primarily job: issue an access token on behalf of a user.

We are going to use the **authorization code grant** flow to obtain a JWT (JSON Web Token). In this flow the user's browser will redirect to the authorization server, where the user will log in and be asked for permissions (or 'scope'). With the user's consent, the authorization server redirect the user back to the client, now with the access token, where it can be used to access the resource by passing it in the ***"Authorization"*** header.

## Dependency
```
dependencies {
     implementation 'org.springframework.boot:spring-boot-starter-oauth2-authorization-server'
}
```

## Using Port 9000
To avoid conflicts, I've changed to use the port 9000.
```
server:
   port: 9000
```

## Configuration
All configuration is in the ***AuthorizationServerConfig*** class.

### Users
To make things simple I went to create two users (in production you'll be going for a database, ldap, so on).
```
@Bean
	public UserDetailsService userDetailsService(PasswordEncoder encoder) {
		UserDetails user1 = User.builder().username("rod").password(encoder.encode("pass")).roles("USER").build();

		UserDetails user2 = User.builder().username("dan").password(encoder.encode("pass")).roles("USER").build();

		return new InMemoryUserDetailsManager(user1, user2);
	}
```

### Authorization Server Defaults
I'm also applying some authorization server defaults. It has *HIGHEST_PRECEDENCE* to make sure if there are more beans of this type declared, this one will take precedence over the others.
```
@Bean
	@Order(Ordered.HIGHEST_PRECEDENCE)
	public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
		OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

		http.getConfigurer(OAuth2AuthorizationServerConfigurer.class).oidc(Customizer.withDefaults());

		return http.formLogin(form -> Customizer.withDefaults()).build();
	}
```
### In-memory Implementation
For demonstration and testing purposes, just go for it. For anything else, you might want to write a custom implementation of RegisteredClientRepository to retrieve client details from a database or somewhere else.
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
A lot going on. In order of appearence:
- creating a random, unique identifier
- defining the name for our client
- client's password
- use of basic authentication (username/password)
- enabling both authorization code and refresh token grants
- redirect URL after authorization has been granted (you can have more than one)
- scopes that the user we'll be able to allow access to.
- without this client settings the scope would be implicitly granted after the user logs in.

Finally, as we're producing JWT tokens, we have a few beans to produce a JWK (JSON Web Key).

## Testing
Pretend to be a client by using this URL in your browser or a curl command-line tool.
http://localhost:9000/oauth2/authorize?response_type=code&client_id=rod-admin-client&redirect_uri=http://localhost:8080/login/oauth2/admin-client&scope=delete+write

You'll be asked to log in, and then be able to choose which permissions (scopes) will be allowed.

As we don't have a client, you'll receive a error. But that's OK, we just want the authorization code, that is in the URL **code** parameter. Now we can ask for an ***access token***!
> curl http://localhost:9000/oauth2/token 
-H "Content-type: application/x-www-form-urlencoded"
-d "grant_type=authorization_code" 
-d "redirect_uri=http://localhost:8080/login/oauth2/admin-client"
-d "code=uEaSvAXma9kaf7ONYCE8gvmHVG751fmLiKR-YbryZNz0HVZrK9uY5oI7Pm3rK82JtWbX9epMQpGSMs5o4cLQEoIanr7MLAnV-KHPw-RJ3nLmhaqnTGaYFRPM3qvzInWQ"
-u rod-admin-client:pass2

You'll receive a JSON responde with the access token, refresh token, and more informations.
>{
  "access_token": "eyJraWQiOiIwYmRkM ...",
  "refresh_token": "vFm86h9e7mbhyhGCO ...",
  "scope": "delete write",
  "token_type": "Bearer",
  "expires_in": 300
}

Now you can use the access token with a header ***"Authorization: Bearer"*** to have access to the client resources.

You can inspect the JWT in the https://jwt.io