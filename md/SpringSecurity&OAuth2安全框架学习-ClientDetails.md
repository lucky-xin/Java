
# 创建ClientDetails信息表sql
```sql
DROP TABLE IF EXISTS `oauth_client_details`;
CREATE TABLE `oauth_client_details` (
  `client_id` varchar(48) NOT NULL,
  `resource_ids` varchar(256) DEFAULT NULL,
  `client_secret` varchar(256) DEFAULT NULL,
  `scope` varchar(256) DEFAULT NULL,
  `authorized_grant_types` varchar(256) DEFAULT NULL,
  `web_server_redirect_uri` varchar(256) DEFAULT NULL,
  `authorities` varchar(256) DEFAULT NULL,
  `access_token_validity` int(11) DEFAULT NULL,
  `refresh_token_validity` int(11) DEFAULT NULL,
  `additional_information` varchar(4096) DEFAULT NULL,
  `autoapprove` varchar(256) DEFAULT NULL,
  PRIMARY KEY (`client_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
```text
默认实现的ClientDetails如下:
表oauth_client_details中, client_id,client_secret为字符串,additionalInformation为json字符串
scope,resource_ids,authorized_grant_types(password,refresh_token,authorization_code,client_credentials其中一个或者多个)，
web_server_redirect_uri,autoapprove为冒号分割的字符串
```
```java
public class BaseClientDetails implements ClientDetails {

	@org.codehaus.jackson.annotate.JsonProperty("client_id")
	@com.fasterxml.jackson.annotation.JsonProperty("client_id")
	private String clientId;

	@org.codehaus.jackson.annotate.JsonProperty("client_secret")
	@com.fasterxml.jackson.annotation.JsonProperty("client_secret")
	private String clientSecret;

	@org.codehaus.jackson.map.annotate.JsonDeserialize(using = JacksonArrayOrStringDeserializer.class)
	@com.fasterxml.jackson.databind.annotation.JsonDeserialize(using = Jackson2ArrayOrStringDeserializer.class)
	private Set<String> scope = Collections.emptySet();

	@org.codehaus.jackson.annotate.JsonProperty("resource_ids")
	@org.codehaus.jackson.map.annotate.JsonDeserialize(using = JacksonArrayOrStringDeserializer.class)
	@com.fasterxml.jackson.annotation.JsonProperty("resource_ids")
	@com.fasterxml.jackson.databind.annotation.JsonDeserialize(using = Jackson2ArrayOrStringDeserializer.class)
	private Set<String> resourceIds = Collections.emptySet();

	@org.codehaus.jackson.annotate.JsonProperty("authorized_grant_types")
	@org.codehaus.jackson.map.annotate.JsonDeserialize(using = JacksonArrayOrStringDeserializer.class)
	@com.fasterxml.jackson.annotation.JsonProperty("authorized_grant_types")
	@com.fasterxml.jackson.databind.annotation.JsonDeserialize(using = Jackson2ArrayOrStringDeserializer.class)
	private Set<String> authorizedGrantTypes = Collections.emptySet();

	@org.codehaus.jackson.annotate.JsonProperty("redirect_uri")
	@org.codehaus.jackson.map.annotate.JsonDeserialize(using = JacksonArrayOrStringDeserializer.class)
	@com.fasterxml.jackson.annotation.JsonProperty("redirect_uri")
	@com.fasterxml.jackson.databind.annotation.JsonDeserialize(using = Jackson2ArrayOrStringDeserializer.class)
	private Set<String> registeredRedirectUris;

	@org.codehaus.jackson.annotate.JsonProperty("autoapprove")
	@org.codehaus.jackson.map.annotate.JsonDeserialize(using = JacksonArrayOrStringDeserializer.class)
	@com.fasterxml.jackson.annotation.JsonProperty("autoapprove")
	@com.fasterxml.jackson.databind.annotation.JsonDeserialize(using = Jackson2ArrayOrStringDeserializer.class)
	private Set<String> autoApproveScopes;

	private List<GrantedAuthority> authorities = Collections.emptyList();

	@org.codehaus.jackson.annotate.JsonProperty("access_token_validity")
	@com.fasterxml.jackson.annotation.JsonProperty("access_token_validity")
	private Integer accessTokenValiditySeconds;

	@org.codehaus.jackson.annotate.JsonProperty("refresh_token_validity")
	@com.fasterxml.jackson.annotation.JsonProperty("refresh_token_validity")
	private Integer refreshTokenValiditySeconds;

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	private Map<String, Object> additionalInformation = new LinkedHashMap<String, Object>();

	public BaseClientDetails() {
	}

	public BaseClientDetails(ClientDetails prototype) {
		this();
		setAccessTokenValiditySeconds(prototype.getAccessTokenValiditySeconds());
		setRefreshTokenValiditySeconds(prototype
				.getRefreshTokenValiditySeconds());
		setAuthorities(prototype.getAuthorities());
		setAuthorizedGrantTypes(prototype.getAuthorizedGrantTypes());
		setClientId(prototype.getClientId());
		setClientSecret(prototype.getClientSecret());
		setRegisteredRedirectUri(prototype.getRegisteredRedirectUri());
		setScope(prototype.getScope());
		setResourceIds(prototype.getResourceIds());
	}

	public BaseClientDetails(String clientId, String resourceIds,
			String scopes, String grantTypes, String authorities) {
		this(clientId, resourceIds, scopes, grantTypes, authorities, null);
	}

	public BaseClientDetails(String clientId, String resourceIds,
			String scopes, String grantTypes, String authorities,
			String redirectUris) {

		this.clientId = clientId;

		if (StringUtils.hasText(resourceIds)) {
			Set<String> resources = StringUtils
					.commaDelimitedListToSet(resourceIds);
			if (!resources.isEmpty()) {
				this.resourceIds = resources;
			}
		}

		if (StringUtils.hasText(scopes)) {
			Set<String> scopeList = StringUtils.commaDelimitedListToSet(scopes);
			if (!scopeList.isEmpty()) {
				this.scope = scopeList;
			}
		}

		if (StringUtils.hasText(grantTypes)) {
			this.authorizedGrantTypes = StringUtils
					.commaDelimitedListToSet(grantTypes);
		} else {
			this.authorizedGrantTypes = new HashSet<String>(Arrays.asList(
					"authorization_code", "refresh_token"));
		}

		if (StringUtils.hasText(authorities)) {
			this.authorities = AuthorityUtils
					.commaSeparatedStringToAuthorityList(authorities);
		}

		if (StringUtils.hasText(redirectUris)) {
			this.registeredRedirectUris = StringUtils
					.commaDelimitedListToSet(redirectUris);
		}
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public String getClientId() {
		return clientId;
	}

	public void setClientId(String clientId) {
		this.clientId = clientId;
	}

	public void setAutoApproveScopes(Collection<String> autoApproveScopes) {
		this.autoApproveScopes = new HashSet<String>(autoApproveScopes);
	}

	@Override
	public boolean isAutoApprove(String scope) {
		if (autoApproveScopes == null) {
			return false;
		}
		for (String auto : autoApproveScopes) {
			if (auto.equals("true") || scope.matches(auto)) {
				return true;
			}
		}
		return false;
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public Set<String> getAutoApproveScopes() {
		return autoApproveScopes;
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public boolean isSecretRequired() {
		return this.clientSecret != null;
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public String getClientSecret() {
		return clientSecret;
	}

	public void setClientSecret(String clientSecret) {
		this.clientSecret = clientSecret;
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public boolean isScoped() {
		return this.scope != null && !this.scope.isEmpty();
	}

	public Set<String> getScope() {
		return scope;
	}

	public void setScope(Collection<String> scope) {
		this.scope = scope == null ? Collections.<String> emptySet()
				: new LinkedHashSet<String>(scope);
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public Set<String> getResourceIds() {
		return resourceIds;
	}

	public void setResourceIds(Collection<String> resourceIds) {
		this.resourceIds = resourceIds == null ? Collections
				.<String> emptySet() : new LinkedHashSet<String>(resourceIds);
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public Set<String> getAuthorizedGrantTypes() {
		return authorizedGrantTypes;
	}

	public void setAuthorizedGrantTypes(Collection<String> authorizedGrantTypes) {
		this.authorizedGrantTypes = new LinkedHashSet<String>(
				authorizedGrantTypes);
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public Set<String> getRegisteredRedirectUri() {
		return registeredRedirectUris;
	}

	public void setRegisteredRedirectUri(Set<String> registeredRedirectUris) {
		this.registeredRedirectUris = registeredRedirectUris == null ? null
				: new LinkedHashSet<String>(registeredRedirectUris);
	}

	@org.codehaus.jackson.annotate.JsonProperty("authorities")
	@com.fasterxml.jackson.annotation.JsonProperty("authorities")
	private List<String> getAuthoritiesAsStrings() {
		return new ArrayList<String>(
				AuthorityUtils.authorityListToSet(authorities));
	}

	@org.codehaus.jackson.annotate.JsonProperty("authorities")
	@org.codehaus.jackson.map.annotate.JsonDeserialize(using = JacksonArrayOrStringDeserializer.class)
	@com.fasterxml.jackson.annotation.JsonProperty("authorities")
	@com.fasterxml.jackson.databind.annotation.JsonDeserialize(using = Jackson2ArrayOrStringDeserializer.class)
	private void setAuthoritiesAsStrings(Set<String> values) {
		setAuthorities(AuthorityUtils.createAuthorityList(values
				.toArray(new String[values.size()])));
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public Collection<GrantedAuthority> getAuthorities() {
		return authorities;
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public void setAuthorities(
			Collection<? extends GrantedAuthority> authorities) {
		this.authorities = new ArrayList<GrantedAuthority>(authorities);
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public Integer getAccessTokenValiditySeconds() {
		return accessTokenValiditySeconds;
	}

	public void setAccessTokenValiditySeconds(Integer accessTokenValiditySeconds) {
		this.accessTokenValiditySeconds = accessTokenValiditySeconds;
	}

	@org.codehaus.jackson.annotate.JsonIgnore
	@com.fasterxml.jackson.annotation.JsonIgnore
	public Integer getRefreshTokenValiditySeconds() {
		return refreshTokenValiditySeconds;
	}

	public void setRefreshTokenValiditySeconds(
			Integer refreshTokenValiditySeconds) {
		this.refreshTokenValiditySeconds = refreshTokenValiditySeconds;
	}

	public void setAdditionalInformation(Map<String, ?> additionalInformation) {
		this.additionalInformation = new LinkedHashMap<String, Object>(
				additionalInformation);
	}

	@org.codehaus.jackson.annotate.JsonAnyGetter
	@com.fasterxml.jackson.annotation.JsonAnyGetter
	public Map<String, Object> getAdditionalInformation() {
		return Collections.unmodifiableMap(this.additionalInformation);
	}

	@org.codehaus.jackson.annotate.JsonAnySetter
	@com.fasterxml.jackson.annotation.JsonAnySetter
	public void addAdditionalInformation(String key, Object value) {
		this.additionalInformation.put(key, value);
	}

	// 省略了toString，equals,hashCode方法
    
}
```