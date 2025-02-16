# Authenticate API requests

The Backstage backend APIs are by default available without authentication. To avoid evil-doers from accessing or modifying data, one might use a network protection mechanism such as a firewall or an authenticating reverse proxy. For Backstage instances that are available on the Internet one can instead use the experimental IdentityClient as outlined below.

API requests from frontend plugins include an authorization header with a Backstage identity token acquired when the user logs in. By adding a middleware that verifies said token to be valid and signed by Backstage, non-authenticated requests can be blocked with a 401 Unauthorized response.

**NOTE**: Enabling this means that Backstage will stop working for guests, as no token is issued for them.

As techdocs HTML pages load assets without an Authorization header the code below also sets a token cookie when the user logs in (and when the token is about to expire).

Create `packages/backend/src/authMiddleware.ts`:

```typescript
import { SingleHostDiscovery } from '@backstage/backend-common';
import type { Config } from '@backstage/config';
import {
  getBearerTokenFromAuthorizationHeader,
  IdentityClient,
} from '@backstage/plugin-auth-node';
import { NextFunction, Request, Response, RequestHandler } from 'express';
import { decodeJwt } from 'jose';
import { URL } from 'url';
import { PluginEnvironment } from './types';

function setTokenCookie(
  res: Response,
  options: { token: string; secure: boolean; cookieDomain: string },
) {
  try {
    const payload = decodeJwt(options.token);
    res.cookie('token', options.token, {
      expires: new Date(payload.exp ? payload.exp * 1000 : 0),
      secure: options.secure,
      sameSite: 'lax',
      domain: options.cookieDomain,
      path: '/',
      httpOnly: true,
    });
  } catch (_err) {
    // Ignore
  }
}

export const createAuthMiddleware = async (
  config: Config,
  appEnv: PluginEnvironment,
) => {
  const discovery = SingleHostDiscovery.fromConfig(config);
  const identity = IdentityClient.create({
    discovery,
    issuer: await discovery.getExternalBaseUrl('auth'),
  });
  const baseUrl = config.getString('backend.baseUrl');
  const secure = baseUrl.startsWith('https://');
  const cookieDomain = new URL(baseUrl).hostname;
  const authMiddleware: RequestHandler = async (
    req: Request,
    res: Response,
    next: NextFunction,
  ) => {
    try {
      const token =
        getBearerTokenFromAuthorizationHeader(req.headers.authorization) ||
        (req.cookies.token as string | undefined);
      if (!token) {
        res.status(401).send('Unauthorized');
        return;
      }
      try {
        req.user = await identity.authenticate(token);
      } catch {
        await appEnv.tokenManager.authenticate(token);
      }
      if (!req.headers.authorization) {
        // Authorization header may be forwarded by plugin requests
        req.headers.authorization = `Bearer ${token}`;
      }
      if (token && token !== req.cookies.token) {
        setTokenCookie(res, {
          token,
          secure,
          cookieDomain,
        });
      }
      next();
    } catch (error) {
      res.status(401).send('Unauthorized');
    }
  };
  return authMiddleware;
};
```

```typescript
// packages/backend/src/index.ts from a create-app deployment

import { createAuthMiddleware } from './authMiddleware';

// ...

async function main() {
  // ...

  const authMiddleware = await createAuthMiddleware(config, appEnv);

  const apiRouter = Router();
  apiRouter.use(cookieParser());
  // The auth route must be publicly available as it is used during login
  apiRouter.use('/auth', await auth(authEnv));
  // Add a simple endpoint to be used when setting a token cookie
  apiRouter.use('/cookie', authMiddleware, (_req, res) => {
    res.status(200).send(`Coming right up`);
  });
  // Only authenticated requests are allowed to the routes below
  apiRouter.use('/catalog', authMiddleware, await catalog(catalogEnv));
  apiRouter.use('/techdocs', authMiddleware, await techdocs(techdocsEnv));
  apiRouter.use('/proxy', authMiddleware, await proxy(proxyEnv));
  apiRouter.use(authMiddleware, notFoundHandler());

  // ...
}
```

Create `packages/app/src/cookieAuth.ts`:

```typescript
import type { IdentityApi } from '@backstage/core-plugin-api';

// Parses supplied JWT token and returns the payload
function parseJwt(token: string): { exp: number } {
  const base64Url = token.split('.')[1];
  const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
  const jsonPayload = decodeURIComponent(
    atob(base64)
      .split('')
      .map(
        c =>
          // eslint-disable-next-line prefer-template
          '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2),
      )
      .join(''),
  );

  return JSON.parse(jsonPayload);
}

// Returns milliseconds until the supplied JWT token expires
function msUntilExpiry(token: string): number {
  const payload = parseJwt(token);
  const remaining =
    new Date(payload.exp * 1000).getTime() - new Date().getTime();
  return remaining;
}

// Calls the specified url regularly using an auth token to set a token cookie
// to authorize regular HTTP requests when loading techdocs
export async function setTokenCookie(url: string, identityApi: IdentityApi) {
  const { token } = await identityApi.getCredentials();
  if (!token) {
    return;
  }

  await fetch(url, {
    mode: 'cors',
    credentials: 'include',
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });

  // Call this function again a few minutes before the token expires
  const ms = msUntilExpiry(token) - 4 * 60 * 1000;
  setTimeout(
    () => {
      setTokenCookie(url, identityApi);
    },
    ms > 0 ? ms : 10000,
  );
}
```

```typescript
// packages/app/src/App.tsx from a create-app deployment

import { setTokenCookie } from './cookieAuth';

// ...

const app = createApp({
  // ...

  components: {
    SignInPage: props => {
      const discoveryApi = useApi(discoveryApiRef);
      return (
        <SignInPage
          {...props}
          providers={['guest', 'custom', ...providers]}
          title="Select a sign-in method"
          align="center"
          onSignInSuccess={async (identityApi: IdentityApi) => {
            setTokenCookie(
              await discoveryApi.getBaseUrl('cookie'),
              identityApi,
            );

            props.onSignInSuccess(identityApi);
          }}
        />
      );
    },
  },

  // ...
});

// ...
```

**NOTE**: Most Backstage frontend plugins come with the support for the `IdentityApi`.
In case you already have a dozen of internal ones, you may need to update those too.
Assuming you follow the common plugin structure, the changes to your front-end may look like:

```diff
// plugins/internal-plugin/src/api.ts
-  import { createApiRef } from '@backstage/core-plugin-api';
+  import { createApiRef, IdentityApi } from '@backstage/core-plugin-api';
import { Config } from '@backstage/config';
// ...

type MyApiOptions = {
    configApi: Config;
+   identityApi: IdentityApi;
    // ...
}

interface MyInterface {
    getData(): Promise<MyData[]>;
}

export class MyApi implements MyInterface {
    private configApi: Config;
+   private identityApi: IdentityApi;
    // ...

    constructor(options: MyApiOptions) {
        this.configApi = options.configApi;
+       this.identityApi = options.identityApi;
    }

    async getMyData() {
        const backendUrl = this.configApi.getString('backend.baseUrl');

+       const { token } = await this.identityApi.getCredentials();
        const requestUrl = `${backendUrl}/api/data/`;
-       const response = await fetch(requestUrl);
+       const response = await fetch(
          requestUrl,
          { headers: { Authorization: `Bearer ${token}` } },
        );
    // ...
   }
```

and

```diff
// plugins/internal-plugin/src/plugin.ts

import {
    configApiRef,
    createApiFactory,
    createPlugin,
+   identityApiRef,
} from '@backstage/core-plugin-api';
import { myPluginPageRouteRef } from './routeRefs';
import { MyApi, myApiRef } from './api';

export const plugin = createPlugin({
    id: 'my-plugin',
    routes: {
        mainPage: myPluginPageRouteRef,
    },
    apis: [
        createApiFactory({
            api: myApiRef,
            deps: {
                configApi: configApiRef,
+               identityApi: identityApiRef,
            },
-           factory: ({ configApi }) =>
-               new MyApi({ configApi }),
+           factory: ({ configApi, identityApi }) =>
+               new MyApi({ configApi, identityApi }),
        }),
    ],
});
```
