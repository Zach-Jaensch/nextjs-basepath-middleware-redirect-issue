# 308 redirects for \_next/data with basePath and middleware

## Get started

```bash
corepack enable
pnpm dev
```

## The issue

This repo has the following set up

- a single pages router page with a link to itself
- middleware (the contents of which doesn't matter)
- basePath configured (any config here will cause the issue)

With the app running, clicking on the link will request the _next/data -> `/library/_next/data/index.json`
This request will get redirected to `/library`, which has a `text/html` format

If you require information from the page props, it will throw as the response is expected to be `application/json`.

## Cause of the issue

Internally, there is a [redirect for trailing slash](https://github.com/vercel/next.js/blob/c3006f6c41484b961919d3526f30370668bbda16/packages/next/src/lib/load-custom-routes.ts#L757-L763), and a [request modifier for next data requests when middleware is present](https://github.com/vercel/next.js/blob/c3006f6c41484b961919d3526f30370668bbda16/packages/next/src/server/lib/router-utils/resolve-routes.ts#L378). 
The request middleware is reformatting the request pathname from `/library/_next/data/index.json` to `/library/` in preparation for the middleware. 
If it was any other page (for example `/test`) the reformatting would change `/library/_next/data/test.json` to `/library/test`.
Without the basePath, this is ok (`/_next/data/index.json` to `/`) however, [with the addition of basePath](https://github.com/vercel/next.js/blob/c3006f6c41484b961919d3526f30370668bbda16/packages/next/src/server/lib/router-utils/resolve-routes.ts#L412-L414), it now has a trailing slash, and hence the trailing slash redirect will be invoked. 

As a workaround, our team has implemented a pnpm patch similar to:
```diff
if (hadBasePath) {
-  normalized = path.posix.join(config.basePath, normalized)
+  normalized = path.posix.join(config.basePath, normalized).replace(/\/$/, "");
}
```