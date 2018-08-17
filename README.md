# hapi-doorkeeper [![Build status for hapi Doorkeeper](https://img.shields.io/circleci/project/sholladay/hapi-doorkeeper/master.svg "Build Status")](https://circleci.com/gh/sholladay/hapi-doorkeeper "Builds")

> User authentication for web servers

This [hapi](https://hapijs.com) plugin makes it easy to add a secure login and logout system for your users by integrating [Auth0](https://auth0.com/).

## Contents

 - [Why?](#why)
 - [Install](#install)
 - [Usage](#usage)
 - [API](#api)
   - [Routes](#routes)
   - [Plugin options](#plugin-options)
 - [Related](#related)
 - [Contributing](#contributing)
 - [License](#license)

## Why?

 - User auth is a necessity for most apps and websites.
 - User auth is difficult to do correctly on your own.
 - Secure systems should be easy to set up and use.
 - Comes with built-in login and logout routes.

## Install

```sh
npm install hapi-doorkeeper --save
```

## Usage

Register the plugin on your server to provide the user auth routes.

```js
const hapi = require('hapi');
const bell = require('bell');
const cookie = require('hapi-auth-cookie');
const doorkeeper = require('hapi-doorkeeper');

const server = hapi.server();

const init = async () => {
    await server.register([bell, cookie, {
        plugin  : doorkeeper,
        options : {
            sessionSecretKey : process.env.SESSION_SECRET_KEY,
            auth0Domain      : process.env.AUTH0_DOMAIN,
            auth0PublicKey   : process.env.AUTH0_PUBLIC_KEY,
            auth0SecretKey   : process.env.AUTH0_SECRET_KEY
        }
    }]);
    server.route({
        method : 'GET',
        path   : '/dashboard',
        config : {
            auth : {
                strategy : 'session',
                mode     : 'required'
            }
        },
        handler(request) {
            const { user } = request.auth.credentials;
            return `Hi ${user.name}, you are logged in! Here is the profile from Auth0: <pre>${JSON.stringify(user.raw, null, 4)}</pre> <a href="/logout">Click here to log out</a>`;
        }
    });
    await server.start();
    console.log('Server ready:', server.info.uri);
};

init();
```

The route above at `/dashboard` can only be accessed by logged in users, as denoted by the `session` strategy being `required`. If you are logged in, it will display your profile, otherwise it will redirect you to a login screen.

Authentication is managed by [Auth0](https://auth0.com/). A few steps are required to finish the integration.

 1. [Sign up for Auth0](https://auth0.com/)
 2. [Set up an Auth0 Application](https://auth0.com/docs/applications/application-types)
 3. [Provide credentials from Auth0](#plugin-options)

After a user logs in, a session cookie is created for them so that the server has a way to remember the user on future requests. The cookie is stateless, encrypted, and secured using flags such as `HttpOnly`. The user's profile is automatically retrieved from Auth0 and stored in the session when they log in. You can access the profile data at `request.auth.credentials.user`. See [hapi-auth-cookie](https://github.com/hapijs/hapi-auth-cookie) and [iron](https://github.com/hueniverse/iron) for details about the cookie implementation and security.

Note that your server must support HTTPS for everything to work properly. If you need help with that, see this [How To Guide](https://medium.freecodecamp.org/how-to-get-https-working-on-your-local-development-environment-in-5-minutes-7af615770eec)

## API

### Routes

Standard user authentication routes are included and will be added to your server when the plugin is registered.

#### GET /login

Tags: `user`, `auth`, `session`, `login`

Begins a user session. If a session is already active, the user will be given the opportunity to log in with a different account.

If the user denies access to a social account, they will be redirected back to the login page so that they may try again, as this usually means they chose the wrong account or provider by accident. All other errors will be returned to the client with a 401 Unauthorized status. You may use [`hapi-error-page`](https://github.com/sholladay/hapi-error-page) or [`onPreResponse`](https://hapijs.com/api#error-transformation) to make beautiful HTML pages for them.

After logging in, the user will be redirected to `/`, the root of the server. To redirect to a different page, send a relative URL in the `next` query parameter (e.g. `GET /login?next=help`).

#### GET /logout

Tags: `user`, `auth`, `session`, `logout`

Ends a user session. Safe to visit regardless of whether a session is active or the validity of the user's credentials. After logging out, the user will be redirected to `/`, the root of the server.

### Plugin options

#### sessionSecretKey

Type: `string`

A passphrase used to secure session cookies. Should be at least 32 characters long and occasionally rotated. See [Iron](https://github.com/hueniverse/iron) for details.

#### auth0Domain

Type: `string`

The domain used to log in to Auth0. This should be the domain of your tenant (e.g. `my-company.auth0.com`) or your own [custom domain](https://auth0.com/docs/custom-domains) (e.g. `auth.my-company.com`).

#### auth0PublicKey

Type: `string`

The ID of your [Auth0 Application](https://manage.auth0.com/#/applications), sometimes referred to as the Client ID.

#### auth0SecretKey

Type: `string`

The secret key of your [Auth0 Application](https://manage.auth0.com/#/applications), sometimes referred to as the Client Secret.

#### providerParams(request)

Type: `function`

An optional event handler where you can decide which query parameters to send to Auth0. Should return an object of key/value pairs that will be serialized to a query string. See the [`providerParams` option](https://github.com/hapijs/bell/blob/master/API.md#options) in [bell](https://github.com/hapijs/bell) for details.

By default, we forward any `screen` parameter passed to `/login`, so that you can implement "Log In" and "Sign Up" buttons that go to the correct screen. To set this up, modify your [Hosted Login Page](https://auth0.com/docs/hosted-pages/login#how-to-customize-your-login-page) and set Lock's [`initialScreen`](https://auth0.com/docs/libraries/lock/v11/configuration#initialscreen-string-) option to use the value of `config.extraParams.screen`. After that, visiting `/login?screen=signUp` will show the Sign Up screen instead of the Log In screen.

#### validateFunc(request, session)

Type: `function`

An optional event handler where you can put business logic to check and modify the session on each request. See the [`validateFunc` option](https://github.com/hapijs/hapi-auth-cookie#hapi-auth-cookie) in [hapi-auth-cookie](https://github.com/hapijs/hapi-auth-cookie) for details.

This is a good place to set [authorization scopes for users](https://futurestud.io/tutorials/hapi-restrict-user-access-with-scopes), if you need to restrict access to some routes for certain users.

## Related

 - [lock](https://github.com/auth0/lock) - UI widget used on the login page

## Contributing

See our [contributing guidelines](https://github.com/sholladay/hapi-doorkeeper/blob/master/CONTRIBUTING.md "Guidelines for participating in this project") for more details.

1. [Fork it](https://github.com/sholladay/hapi-doorkeeper/fork).
2. Make a feature branch: `git checkout -b my-new-feature`
3. Commit your changes: `git commit -am 'Add some feature'`
4. Push to the branch: `git push origin my-new-feature`
5. [Submit a pull request](https://github.com/sholladay/hapi-doorkeeper/compare "Submit code to this project for review").

## License

[MPL-2.0](https://github.com/sholladay/hapi-doorkeeper/blob/master/LICENSE "License for hapi-doorkeeper") © [Seth Holladay](https://seth-holladay.com "Author of hapi-doorkeeper")

Go make something, dang it.
