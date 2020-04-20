---
lang: en
title: 'Migrating Passport-based authentication'
keywords: LoopBack 4.0, LoopBack 4, LoopBack 3, Migration
sidebar: lb4_sidebar
permalink: /doc/en/lb4/migration-auth-passport.html
---

{% include tip.html content="
Missing instructions for your LoopBack 3 use case? Please report a [Migration docs issue](https://github.com/strongloop/loopback-next/issues/new?labels=question,Migration,Docs&template=Migration_docs.md) on GitHub to let us know.
" %}

# Migrating Passport Strategies 

This page guides on migrating lb3 apps that used [Passport Strategies](http://www.passportjs.org/) for authentication usecases. Before following this guide, please know more about the [@loopback/authentication-passport](https://loopback.io/doc/en/lb4/Authentication-passport.html) package.


## Differences between LoopBack 3 and LoopBack 4

In LoopBack 3, routes can be configured explicitly as authentication providers using express style passport strategies middleware. Also the [lb3 passport component](https://github.com/strongloop/loopback-component-passport) helped with implicit
authentication configuration using json files. It had built-in model classes to search users and persist user identities.

In LoopBack 4, authentication endpoints are configured in controllers and the `@authenticate` decorator tells which passport
strategy to configure for that API route. Also the [@loopback/authentication-passport](https://loopback.io/doc/en/lb4/Authentication-passport.html) package is necessary to bridge between passport strategies and the authentication design of lb4.

## An example passport login app

To demonstrate how to implement passport strategies in LoopBack 4 and migrate LB3 apps using[loopback-component-passport](https://github.com/strongloop/loopback-component-passport), a [passport-login](https://github.com/strongloop/loopback-next/tree/master/examples/passport-login) example app is now available.

This example is migrated from
[loopback-example-passport](https://github.com/strongloop/loopback-example-passport),
It demonstrates how to use the LoopBack 4 features (like @authenticate decorator, strategy providers, etc) with passport strategies. It includes the oauth2 strategies to interact with external OAuth providers like facebook, google,etc as well as local and basic strategies.

You can use this example to see how to,

- Log in or Sign up into a LoopBack App using passport strategy modules
- Log in via external apps like Facebook or link those external profiles with a LoopBack user (for example, a LoopBack user can have associated facebook/google accounts to retrieve pictures).
- Use basic or local passport strategy modules

We divide this guide into two sections: 
  1. [How to migrate OAuth Strategies ?](#OAuth2-Strategies)
  2. [How to migrate Non-OAuth, ie, Strategies like basic, local, etc ?](#Non-OAuth2-Strategies)

In each of these sections the following are explained :

A. Configuring Authentication Endpoints :
    Authentication endpoints are controller methods that validate user credentials and provides the caller with a login session which is usually represented by an access token or a cookie.

B. Strategy Providers :
     In LoopBack4 passport strategies will have to be injected into the authentication using provider classes. 

## OAuth2 Strategies

### Configuring Authentication endpoints in controllers

Authentication endpoints provide functions that validate user credentials and provides a login session which is usually represented by an access token or a cookie.

```ts
  @authenticate('oauth2')
  @get('/auth/thirdparty/{provider}')
  /**
   * Endpoint: '/auth/thirdparty/{provider}'
   *          an endpoint for api clients to login via a third party app, redirects to third party app
   */
  loginToThirdParty(
    @param.path.string('provider') provider: string,
    @inject(AuthenticationBindings.AUTHENTICATION_REDIRECT_URL)
    redirectUrl: string,
    @inject(AuthenticationBindings.AUTHENTICATION_REDIRECT_STATUS)
    status: number,
    @inject(RestBindings.Http.RESPONSE)
    response: Response,
  ) {
    response.statusCode = status || 302;
    response.setHeader('Location', redirectUrl);
    response.end();
    return response;
  }

  @authenticate('oauth2')
  @get('/auth/thirdparty/{provider}/callback')
  /**
   * Endpoint: '/auth/thirdparty/{provider}/callback'
   *          an endpoint which serves as a oauth2 callback for the thirdparty app
   *          this endpoint sets the user profile in the session
   */
  async thirdPartyCallBack(
    @param.path.string('provider') provider: string,
    @inject(SecurityBindings.USER) user: UserProfile,
    @inject(RestBindings.Http.REQUEST) request: RequestWithSession,
    @inject(RestBindings.Http.RESPONSE) response: Response,
  ) {
    const profile = {
      ...user.profile,
    };
    request.session.user = profile;
    response.redirect('/auth/account');
    return response;
  }
```

### Models

```ts
@model()
export class UserIdentity extends Entity {
  @property({
    type: 'string',
    id: true,
  })
  id: string;

  @property({
    type: 'string',
    required: true,
  })
  provider: string;

  @property({
    type: 'object',
    required: true,
  })
  profile: object;

  @property({
    type: 'object',
  })
  credentials?: object;

  @property({
    type: 'string',
    required: true,
  })
  authScheme: string;

  @property({
    type: 'date',
    required: true,
  })
  created?: Date;

  @belongsTo(() => User)
  userId: number;

  constructor(data?: Partial<UserIdentity>) {
    super(data);
  }
}
```


## Non OAuth2 Strategies

```ts
  @authenticate('session')
  @get('/whoAmI', {
    responses: USER_PROFILE_RESPONSE,
  })
  whoAmI(@inject(SecurityBindings.USER) user: UserProfile): object {
    /**
     * controller returns back currently logged in user information
     */
    return {
      user: user.profile,
      headers: Object.assign({}, this.req.headers),
    };
  }
```

```ts
  @authenticate('basic')
  @get('/profiles')
  async getExternalProfiles(
    @inject(SecurityBindings.USER) profile: UserProfile,
  ) {
    const user = await this.userRepository.findById(
      parseInt(profile[securityId]),
      {
        include: [
          {
            relation: 'profiles',
          },
        ],
      },
    );
    return user.profiles;
  }
```
