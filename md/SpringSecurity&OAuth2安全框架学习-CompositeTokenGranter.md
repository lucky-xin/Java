# CompositeTokenGranter实现原理，遍历所有的TokenGranter实现类如果某个类处理之后返回OAuth2AccessToken不为null,则直接返回此OAuth2AccessToken
```java
package org.springframework.security.oauth2.provider;

import java.util.ArrayList;
import java.util.List;

import org.springframework.security.oauth2.common.OAuth2AccessToken;

/**
 * @author Dave Syer
 * 
 */
public class CompositeTokenGranter implements TokenGranter {

	private final List<TokenGranter> tokenGranters;

	public CompositeTokenGranter(List<TokenGranter> tokenGranters) {
		this.tokenGranters = new ArrayList<TokenGranter>(tokenGranters);
	}
	
	public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
		for (TokenGranter granter : tokenGranters) {
			OAuth2AccessToken grant = granter.grant(grantType, tokenRequest);
			if (grant!=null) {
				return grant;
			}
		}
		return null;
	}
	
	public void addTokenGranter(TokenGranter tokenGranter) {
		if (tokenGranter == null) {
			throw new IllegalArgumentException("Token granter is null");
		}
		tokenGranters.add(tokenGranter);
	}

}
```