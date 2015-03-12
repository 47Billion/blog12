title: "Securing, Versioning and Auditing REST (JAX-RS, Jersey) APIs"
tags:
  - Java
  - JAX-RS
  - Jersey
  - jsr250
  - REST
  - Security
id: 1903
categories:
  - Technology
date: 2012-03-02 19:19:33
---

Now that your functionalities are working, you want a layer of security to authenticate/authorize your APIs. Though this is a bad approach towards security, but – I know – real life is a tough game and nothing happens they way they should be... and so be it. Additionally, you might want to control API versions (i.e. expose newer APIs only to newer clients) and audit API usage.

Well, I am going to propose a _tangential_ way to implement these concerns. You won't need to touch any of your business logic as such. Only few annotations (custom and otherwise) would need to be applied. That way, you won't feel bad about missing these things when you started the project and your concerns will be taken care of in the most un-obtrusive way possible. Wonderful... eh?

First, you will need to create some sort of **sign-in** API, which will accept username/password (or oAuth or whatever you fancy) and generate some sort of session information which you will store in some database ([Redis](http://redis.io/) maybe!) and share its ID, say _sessionId_, with client. Then, with every subsequent request, Client will attach this sessionId in the request header which server will pick and look up for associated session information (permission, roles etc.) and based upon that server will authenticate and authorize the request. 

Here is your Session bean.  

```java
package com.strumsoft.api;

import java.io.Serializable;
import java.util.Date;

/**
 * @author &quot;Animesh Kumar &lt;animesh@strumsoft.com&gt;&quot;
 *
 */
public class Session implements Serializable {
	// 
	private static final long serialVersionUID = -7483170872690892182L;

	private String sessionId;   // id
	private String userId;      // user
	private boolean active;     // session active?
	private boolean secure;     // session secure?

	private Date createTime;    // session create time
	private Date lastAccessedTime;  // session last use time

	// getters/setters here
}
```

And this is your User bean. This must implement [java.security.Principal](http://docs.oracle.com/javase/6/docs/api/java/security/Principal.html). 

```java

package com.strumsoft.api;

import java.util.Set;

/**
 * @author &quot;Animesh Kumar &lt;animesh@strumsoft.com&gt;&quot;
 *
 */
public class User implements java.security.Principal {
	// Role
	public enum Role {
		Editor, Visitor, Contributor
	};

	private String userId;          // id
	private String name;            // name
	private String emailAddress;    // email
	private Set&lt;Role&gt; roles;        // roles

	@Override
	public String getName() {
		return null;
	}

	// getters/setters here
}
```

Now, you need to implement [javax.ws.rs.core.SecurityContext](http://jackson.codehaus.org/javadoc/jax-rs/1.0/javax/ws/rs/core/SecurityContext.html). This will be bound to incoming request and will decide whether to allow or deny it.  

```java
package com.strumsoft.api;

import java.security.Principal;
import javax.ws.rs.WebApplicationException;
import javax.ws.rs.core.Response;
import javax.ws.rs.core.SecurityContext;

/**
 * @author &quot;Animesh Kumar &lt;animesh@strumsoft.com&gt;&quot;
 *
 */
public class MySecurityContext implements javax.ws.rs.core.SecurityContext {

	private final User user;
	private final Session session;

	public MySecurityContext(Session session, User user) {
		this.session = session;
		this.user = user;
	}

	@Override
	public String getAuthenticationScheme() {
		return SecurityContext.BASIC_AUTH;
	}

	@Override
	public Principal getUserPrincipal() {
		return user;
	}

	@Override
	public boolean isSecure() {
		return (null != session) ? session.isSecure() : false;
	}

	@Override
	public boolean isUserInRole(String role) {

		if (null == session || !session.isActive()) {
			// Forbidden
			Response denied = Response.status(Response.Status.FORBIDDEN).entity(&quot;Permission Denied&quot;).build();
			throw new WebApplicationException(denied);
		}

		try {
			// this user has this role?
			return user.getRoles().contains(User.Role.valueOf(role));
		} catch (Exception e) {
		}

		return false;
	}
}
```

Then, you need a [ResourceFilter](http://jersey.java.net/nonav/apidocs/latest/jersey/com/sun/jersey/spi/container/ResourceFilter.html) which will intercept the request, look for sessionId in the header and generate and attach our SecurityContext implementation to it. Notice, how our implementation only gets applied on Request but not on Response. 

```java
package com.strumsoft.api;

import javax.ws.rs.ext.Provider;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import com.flockthere.api.repository.SessionRepository;
import com.flockthere.api.repository.UserRepository;
import com.sun.jersey.spi.container.ContainerRequest;
import com.sun.jersey.spi.container.ContainerRequestFilter;
import com.sun.jersey.spi.container.ContainerResponseFilter;
import com.sun.jersey.spi.container.ResourceFilter;

/**
 * Filter all incoming requests, look for possible session information and use that
 * to create and load a SecurityContext to request. 
 * @author &quot;Animesh Kumar &lt;animesh@strumsoft.com&gt;&quot;
 * 
 */
@Component   // let spring manage the lifecycle
@Provider    // register as jersey's provider
public class SecurityContextFilter implements ResourceFilter, ContainerRequestFilter {

	@Autowired
	private SessionRepository sessionRepository;  // DAO to access Session

	@Autowired
	private UserRepository userRepository;  // DAO to access User

	@Override
	public ContainerRequest filter(ContainerRequest request) {
		// Get session id from request header
		final String sessionId = request.getHeaderValue(&quot;session-id&quot;);

		User user = null;
		Session session = null;

		if (sessionId != null &amp;&amp; sessionId.length() &gt; 0) {
			// Load session object from repository
			session = sessionRepository.findOne(sessionId);

			// Load associated user from session
			if (null != session) {
				user = userRepository.findOne(session.getUserId());
			}
		}

		// Set security context
		request.setSecurityContext(new MySecurityContext(session, user));
		return request;
	}

	@Override
	public ContainerRequestFilter getRequestFilter() {
		return this;
	}

	@Override
	public ContainerResponseFilter getResponseFilter() {
		return null;
	}
}
```

Okay, the hard part is over. All we need now is a way to fire our SecurityContextFilter. For this, we will create a [ResourceFilterFactory](http://jersey.java.net/nonav/apidocs/latest/jersey/com/sun/jersey/spi/container/ResourceFilterFactory.html). During application startup, this factory will create a List of filters for all [AbstractMethod](http://jersey.java.net/nonav/apidocs/latest/jersey/com/sun/jersey/api/model/AbstractMethod.html)s of each of our Resources. We are going to extend [RolesAllowedResourceFilterFactory](http://jersey.java.net/nonav/apidocs/latest/jersey/com/sun/jersey/api/container/filter/RolesAllowedResourceFilterFactory.html) which will generate all Role based ResourceFilters for us. And then, we will add our SecurityContextFilter on the top of the list with VersionFilter and AuditFilter in the bottom. That way, SecurityContextFilter will executed first because you need to make auth decisions early. VersionFilter will be next. And Audit in the bottom. You want to audit when everything else has been done. No? 

```java
package com.strumsoft.api;

import java.util.ArrayList;
import java.util.List;

import javax.ws.rs.ext.Provider;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import com.flockthere.api.AllowAllVersions;
import com.flockthere.api.audit.Audit;
import com.flockthere.api.resource.interceptor.AuditingFilter;
import com.flockthere.api.resource.interceptor.SecurityContextFilter;
import com.flockthere.api.resource.interceptor.VersionFilter;
import com.sun.jersey.api.container.filter.RolesAllowedResourceFilterFactory;
import com.sun.jersey.api.model.AbstractMethod;
import com.sun.jersey.spi.container.ResourceFilter;

/**
 * FilterFactory to create List of request/response filters to be applied on a particular
 * AbstractMethod of a resource.
 * 
 * @author &quot;Animesh Kumar &lt;animesh@strumsoft.com&gt;&quot;
 * 
 */
@Component // let spring manage the lifecycle
@Provider  // register as jersey's provider
public class ResourceFilterFactory extends RolesAllowedResourceFilterFactory {

	@Autowired
	private SecurityContextFilter securityContextFilter;

	// Similar to SecurityContextFilter to check incoming requests for API Version information and
	// act accordingly
	@Autowired
	private VersionFilter versionFilter;

	// Similar to SecurityContextFilter to audit incoming requests
	@Autowired
	private AuditingFilter auditingFilter;

	@Override
	public List&lt;ResourceFilter&gt; create(AbstractMethod am) {
		// get filters from RolesAllowedResourceFilterFactory Factory!
		List&lt;ResourceFilter&gt; rolesFilters = super.create(am);
		if (null == rolesFilters) {
			rolesFilters = new ArrayList&lt;ResourceFilter&gt;();
		}

		// Convert into mutable List, so as to add more filters that we need
		// (RolesAllowedResourceFilterFactory generates immutable list of filters)
		List&lt;ResourceFilter&gt; filters = new ArrayList&lt;ResourceFilter&gt;(rolesFilters);

		// Load SecurityContext first (this will load security context onto request)
		filters.add(0, securityContextFilter);

		// Version Control?
		filters.add(versionFilter);

		// If this abstract method is annotated with @Audit, we will apply AuditFilter to audit
		// this request.
		if (am.isAnnotationPresent(Audit.class)) {
			filters.add(auditingFilter);
		}

		return filters;
	}
}
```

VersionFilter will help control Client's access to API methods based upon client's version. Implementation would be similar to SecurityContextFilter. You will need to annotate API methods with @Version("&gt;1.3"). VersionFilter will read this value ("&gt;1.3"), check request headers for API-Version keys and then decide whether to allow or reject the request. Similarly, AuditFilter will log all such annotated (@Audit("audit-note")) API methods. I will not discuss their actual implementations. You can very easily write them based upon your requirement or can remove them altogether if not needed. 

Here is how these annotations will look like. 

```java
// @Version
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD })
public @interface Version {
	String value();
}

// @Audit
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD })
public @interface Audit {
	String value();
}
```

And now, you will need to update you web.xml to include ResourceFilterFactory 

```xml
<servlet>
    <servlet-name>REST</servlet-name>
    <servlet-class>
        com.sun.jersey.spi.spring.container.servlet.SpringServlet
    </servlet-class>
    ...
    <!-- Our resource filter for tangential concerns (security, logging, version etc.) -->
    <init-param>
        <param-name>com.sun.jersey.spi.container.ResourceFilters</param-name>
        <param-value>com.strumsoft.api.ResourceFilterFactory</param-value>
    </init-param>
    ...
</servlet>
```

That's it. All configuration is done. Now, you just need to annotate you Resources. Let's see how. 

Say, you have a BookResource which exposes APIs to create/edit/list books. And you want 
1\. "Editor/Contributor" to be able to create a book, (**@RolesAllowed({ "Editor", "Contributor" })**)
2\. Only "Editor" to be able to edit a book, (**@RolesAllowed({ "Editor" })**)
3\. While, anyone can see all the books. (**@PermitAll**)

Also, let's say editing a book was released in API 1.3, so any client still using older APIs should not be able to update (**@Version("&gt;1.3")**) an existing book. (Assuming you implemented VersionFilter properly)

Additionally, create or edit book should be audited with respective notes "create-book" and "edit-book" given your AuditFilter is in place. 

```java
package com.strumsoft.api;

import javax.ws.rs.FormParam;
import javax.ws.rs.GET;
import javax.ws.rs.POST;
import javax.ws.rs.PUT;
import javax.ws.rs.Produces;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.SecurityContext;

import com.flockthere.api.audit.Audit;

/**
 * Book Resource
 * 
 * @Path(&quot;/book/&quot;)
 * @author &quot;Animesh Kumar &lt;animesh@strumsoft.com&gt;&quot;
 * 
 */
@Produces({ MediaType.APPLICATION_XML, MediaType.APPLICATION_JSON })
public interface BookResource {

	// Only Editor and Contributor can create a book entry
	@javax.annotation.security.RolesAllowed({ &quot;Editor&quot;, &quot;Contributor&quot; })
	@Audit(&quot;create-book&quot;)  // Audit
	@POST
	public Book createBook(@Context SecurityContext sc, @FormParam(&quot;title&quot;) String title,
			@FormParam(&quot;author&quot;) String author, @FormParam(&quot;publisher&quot;) String publisher,
			@FormParam(&quot;isbn&quot;) String isbn, @FormParam(&quot;edition&quot;) String edition);

	// Only Editor can edit an existing book entry
	@javax.annotation.security.RolesAllowed({ &quot;Editor&quot; })
	@Audit(&quot;edit-book&quot;)   // Audit
	@Version(&quot;&gt;1.3&quot;)  // Available only on API version 1.3 onwards
	@PUT
	public Book editBook(@Context SecurityContext sc, @FormParam(&quot;title&quot;) String title,
			@FormParam(&quot;author&quot;) String author, @FormParam(&quot;publisher&quot;) String publisher,
			@FormParam(&quot;isbn&quot;) String isbn, @FormParam(&quot;edition&quot;) String edition);

	// Anyone can see these books
	@javax.annotation.security.PermitAll
	@GET
	public Book listBooks();

}
```

And at last, you will need to add [jsr250-api](http://jcp.org/en/jsr/detail?id=250) to your project dependencies which defines javax.annotation.security.* annotations. 

```xml
<!-- securty tags: javax.annotation.security.* (@RolesAllowed, @PermitAll, @DenyAll etc.) -->
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>jsr250-api</artifactId>
    <version>1.0</version>
</dependency>
```

Chill. :)