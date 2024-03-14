---
description: >-
  Spring Security 설정에서 `AnyRequest().authenticated()`로 설정했음에도 불구하고 엔드포인트에 접근이
  가능하던 이슈
---

# AnonymousUser 접근 가능 이슈

스프링 시큐리티에 별도의 설정을 하지 않고  아래와 같이 설정하면 PERMIT\_URL을 제외한 모든 요청은 인증을 완료해야 접근이 가능하다는 의미로 사용된다.

```java
http.authorizeRequest()
    .antMatchers(PERMIT_URL)
    .permitAll()
    .anyRequest().authenticated();
```

따라서 인증이 되지 않은 요청에 대해서는 401 에러가 발생하게 된다.

문제가 되었던 상황은 Controller에 특정 어노테이션이 붙어있는 엔드포인트들에 대해서 접근 권한을 부여하려고 `FilterInvocationSecurityMetadataSource`를 구현한 구현체를 인터셉터를 추가 한 상황이였다.

```java
http.addFilterBefore(CUSTOM_INTERCEPTOR_IMPL(), FilterSecurityInterceptor.class)
```

인터셉터를 설정하지 않았다면 `DefaultFilterInvocationSecurityMetadataSource.java`에서  위에서 설정한 anyRequest에 대한 설정이 ConfigAttribute에 "authenticated"란 값을 가지고 리턴을 하게 되고&#x20;

```java
@Override
public Collection<ConfigAttribute> getAttributes(Object object) {
	final HttpServletRequest request = ((FilterInvocation) object).getRequest();
	int count = 0;
	for (Map.Entry<RequestMatcher, Collection<ConfigAttribute>> entry : this.requestMap.entrySet()) {
		if (entry.getKey().matches(request)) {
			return entry.getValue();
		}
		else {
			if (this.logger.isTraceEnabled()) {
				this.logger.trace(LogMessage.format("Did not match request to %s - %s (%d/%d)", entry.getKey(),
						entry.getValue(), ++count, this.requestMap.size()));
			}
		}
	}
	return null;
}
```

`AbstractSecurityInterceptor.attemptAuthorization()`에서 설정한 접근 인가 방식(AffirmativeBase(), ConsensusBase(), UnanimousBased())에서 Voter들의 투표를 거쳐서 인가를 결정하게 되고 인가되지 않은 요청은 예외를 발생시켜 권한이 없는 사용자는 접근을 하지 못하게 된다.

```java
private void attemptAuthorization(Object object, Collection<ConfigAttribute> attributes,
		Authentication authenticated) {
	try {
		this.accessDecisionManager.decide(authenticated, object, attributes);
	}
	catch (AccessDeniedException ex) {
		if (this.logger.isTraceEnabled()) {
			this.logger.trace(LogMessage.format("Failed to authorize %s with attributes %s using %s", object,
					attributes, this.accessDecisionManager));
		}
		else if (this.logger.isDebugEnabled()) {
			this.logger.debug(LogMessage.format("Failed to authorize %s with attributes %s", object, attributes));
		}
		publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated, ex));
		throw ex;
	}
}
```



하지만 커스텀한 구현체를 사용하면서 특정 어노테이션이 붙은 엔드포인트들에 대해서만 권한을 체크하고 어노테이션이 붙지 않는 엔드포인트는 null을 반환하게 설정이 되어 있었다.  null을 반환하게 되면 `attemptAuthorization()`의 로직이 아닌 다른 로직을 타게 되고 스프링 시큐리티 기본정책에 의해 AnonymousUser의 권한을 부여 받게 된다. 그럼 사용자는 AnonymousUser의 권한으로 엔드포인트에 접근이 가능하게 되었던 것이였다.

따라서 특정 어노테이션에 대한 접근을 제외한 모든요청에 "authenticated"란 값을 응답하므로써 인증이 필요한 요청이라는 것을 반환하면 권한이 없는 요청에 대해서도 접근이 되지 않도록 할 수 있다.

