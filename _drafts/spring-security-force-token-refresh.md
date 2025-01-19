---
layout: post
title: "Forcing an OAuth2/OpenID Connect token refresh using Spring Security"
categories: software
tags: [java, keycloak, oauth2, openid-connect, spring, spring-security, workaround]
---

Hello there, and welcome to my first ever post on this blog! Let's jump right into an issue I recently faced in hopes the solution may be useful to someone else.

For the past few months I've been working on a side project of mine --- a [Spring Boot][spring-boot] web application --- which implements authentication by outsourcing the task to [Keycloak][keycloak] as OpenID provider, with Keycloak being integrated into my application using [Spring Security][spring-security]. As far as architectures go, this may be one of the more common ways to leverage OpenID Connect on the JVM.

Besides authentication, Keycloak (realm) roles were also used for simple, role-based access control by mapping them to Spring Security [`GrantedAuthorities`][granted-authority] using a [`GrantedAuthoritiesMapper`][granted-authorities-mapper]. There only ever was a single role which users were either assigned or not. A webhook event from a third-party service was responsible for granting the role to existing users.

Eventually, I decided to decouple the webhook event receival and role assignment logic out of my application and into a Keycloak extension, the idea being that a Keycloak extension was more reusable across projects --- a decision I regretted a while later and ended up reversing, actually. However, at that time every aspect of my solution looked quite elegant: the Spring Boot application was no longer responsible for the authorization logic save for checking if the currently authenticated principal had the correct role. All of the other logic was now contained within Keycloak, but unfortunately it wouldn't stay so simple and straightforward.

Since the application no longer got notice of the webhook event, it wouldn't be able to update the sessions of users any more if they were assigned the new role, thus forcing them to sign out and back in if they wanted to use their new privileges, refreshing the access and ID tokens obtained from Keycloak as part of the login process. A quite bad user experience in my opinion. A solution was required: the ability to force a token refresh, which I expected to be easy to find on StackOverflow or blog posts like mine, but it didn't seem like anyone else already had this requirement. After investing a bit of time searching I decided to tackle the issue on my own.

I first needed a kind of "hook" to know when a refresh of the tokens might be necessary for a particular user. An [`AccessDeniedHandler`][access-denied-handler] came to mind because the application is a traditional <abbr title="Multi-page application">MPA</abbr> using server-side rendering and the only permission granted by my sole application role was the ability to access specific endpoints and resources. Navigating to a locked endpoint would trigger an `AccessDeniedException` which could then be handled by an `AccessDeniedHandler` in order to refresh the tokens and finally retry the original request. This is what I came up with:

{% highlight java %}
import java.io.IOException;
import java.net.URI;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;

public class OidcTokenForceRefreshingAccessDeniedHandler
        implements AccessDeniedHandler {

    private final OidcTokenForceRefresher oidcTokenForceRefresher;

    private static final String HANDLED_ATTRIBUTE_NAME =
        OidcTokenForceRefreshingAccessDeniedHandler.class.getName() + ".handled";

    public OidcTokenForceRefreshingAccessDeniedHandler(
            OidcTokenForceRefresher oidcTokenForceRefresher) {
        this.oidcTokenForceRefresher = oidcTokenForceRefresher;
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       AccessDeniedException accessDeniedException)
            throws IOException, ServletException {
        var handled = (Boolean) request.getAttribute(HANDLED_ATTRIBUTE_NAME);
        if ((handled != null) && handled)
            return;
        request.setAttribute(HANDLED_ATTRIBUTE_NAME, true);

        oidcTokenForceRefresher.forceTokenRefresh(request, response);

        final var requestPath = URI.create(request.getRequestURI()).getPath();
        request.getRequestDispatcher(requestPath).forward(request, response);
    }
}
{% endhighlight %}

The request attribute `HANDLED_ATTRIBUTE_NAME` is needed to prevent the `AccessDeniedHandler` from running into an endless loop when the tokens have been refreshed and the request is retried, but the user still isn't able to access the originally requested resource. Forcing a token refresh is delegated to a separate class, the adequately named `OidcTokenForceRefresher`. Once the tokens have been refreshed the `RequestDispatcher`'s `forward` method is used to retry the request on the server-side, avoiding the need for a client-side redirect.

Now, the real fun begins with the implementation of `OidcTokenForceRefresher`:

{% highlight java %}
import org.springframework.stereotype.Component;

@Component
public class OidcTokenForceRefresher {

    public void forceTokenRefresh(HttpServletRequest request,
                                  HttpServletResponse response) {
    }
}
{% endhighlight %}

[keycloak]: https://www.keycloak.org
[spring-boot]: https://spring.io/projects/spring-boot
[spring-security]: https://spring.io/projects/spring-security
[granted-authority]: https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/core/GrantedAuthority.html
[granted-authorities-mapper]: https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/core/authority/mapping/GrantedAuthoritiesMapper.html
[access-denied-handler]: https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/web/access/AccessDeniedHandler.html
