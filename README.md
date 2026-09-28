# Integration of SEO4Ajax in Shopify

[SEO4Ajax](https://www.seo4ajax.com) is a service that allows AJAX websites
(e.g. based on React, Vue.js, Svelte, Angular, Backbone, Ember, jQuery etc.) to
be indexable by search engines, social and ad networks.

This file describes how to integrate SEO4Ajax in a [Shopify](https://www.shopify.com)
store, through a Shopify
[app proxy](https://shopify.dev/docs/apps/build/online-store/app-proxies).

## How it works

A Shopify storefront cannot be placed behind a proxy of your own: Shopify serves it
from its own CDN and terminates TLS for the shop's domain. The only supported way to
run your own code on a storefront URL, and to decide what HTML that URL returns, is an
app proxy. Shopify forwards requests from `https://<shop>/apps/<subpath>` to a route in
your Shopify app and returns your response to the visitor, which is where the choice
between a prerendered snapshot and the client-side page is made.

Two consequences follow:

- Only pages served through the app proxy can be prerendered. Product, collection and
  regular pages have no request hook.
- Those pages live under a fixed prefix. `prefix` must be one of `a`, `apps`,
  `community` or `tools`, and `subpath` is a single segment of letters, digits,
  underscores and hyphens. A page reachable at `https://<shop>/apps/my-page` is
  possible; one at `https://<shop>/support/my-page` is not. When you are moving a
  JavaScript-rendered page onto Shopify, redirect its former address to the new one.

The route also re-serves the sitemap that SEO4Ajax builds from the URLs it has
prerendered, at `https://<shop>/apps/my-page/sitemap.xml`. Unlike a snapshot it goes to
every visitor, crawler or not: it is one document, the same for all of them, and a
sitemap only a crawler can reach is a sitemap nobody can debug.

## Requirements

- A Shopify app. The route below uses the React Router app template and
  `@shopify/shopify-app-react-router`; `@shopify/shopify-app-remix` exposes the same
  `authenticate.public.appProxy` API.
- The site token from the site settings page in the console of SEO4Ajax
  (e.g. `12918fa3c2e945aaf4625a81c93a3181`).

## Instructions

1. Declare the app proxy in `shopify.app.toml`, and add the `write_app_proxy` scope:

   ```toml
   [access_scopes]
   scopes = "write_app_proxy" # alongside the scopes your app already requests

   [app_proxy]
   url = "/seo4ajax-proxy"
   prefix = "apps"
   subpath = "my-page"
   ```

   `url` may be relative to `application_url`, as above, or absolute. The page is then
   served at `https://<shop>/apps/my-page`, and `https://<shop>/apps/my-page/child`
   reaches `<application_url>/seo4ajax-proxy/child`.

2. Add the site token to your app's environment, as `SEO4AJAX_SITE_TOKEN`.

3. Create the route at `app/routes/seo4ajax-proxy.jsx`, matching the `url` you
   configured, and paste the code below.

4. Replace the contents of `PAGE` with your own page: the element your client-side
   application mounts into, and the script that renders it.

5. Announce the sitemap to crawlers. Nothing links to it, so add a `Sitemap` directive
   at the end of the theme's `robots.txt.liquid` (Online Store > Themes > … > Edit code,
   creating the template if it does not exist yet):

   ```liquid
   {%- comment -%} Liquid for the default rules, then: {%- endcomment -%}

   Sitemap: https://<shop>/apps/my-page/sitemap.xml
   ```

   Shopify's own sitemap stays where it is; a host may declare more than one. Do this
   only once the SEO4Ajax site is registered for the storefront's own domain — see
   **The sitemap** below.

6. Deploy the app with `shopify app deploy`, then **install it on the shop, or
   reinstall it if it is already installed**. Changes to `prefix` and `subpath` only
   take effect for new installations.

7. Check the result, with and without a crawler user agent:

   ```sh
   curl -A "Googlebot" https://<shop>/apps/my-page  # the prerendered snapshot
   curl https://<shop>/apps/my-page                 # the client-side page
   curl -i https://<shop>/apps/my-page/sitemap.xml  # the sitemap, for either of them
   ```

## Code

```js
import { authenticate } from "../shopify.server";

const SITE_TOKEN = process.env.SEO4AJAX_SITE_TOKEN;
const API_URL = "https://api.seo4ajax.com/";

const USER_AGENT_TEST = /google|bot|-user|meta-external|lighthouse|spider|pinterest|crawler|archiver|flipboardproxy|mediapartners|facebookexternalhit|insights|quora|whatsapp|slurp/i;
const FILENAME_EXTENSION_TEST = /\.(?!htm)[^.]+$/;

// SEO4Ajax's own scraper, which renders the page and must therefore always receive the
// client-side version, or prerendering would recurse into itself.
const SCRAPER_TEST = /\bs4a\//i;

// Shopify appends these to every proxied request. timestamp and signature change on
// each one, so leaving them in the snapshot URL would miss the cache every time.
const SHOPIFY_PROXY_PARAMS = [
    "shop",
    "path_prefix",
    "timestamp",
    "signature",
    "logged_in_customer_id",
];

// Must match app_proxy.url in shopify.app.toml.
const APP_PATH = "/seo4ajax-proxy";

// A cold render must not blank the page for a real crawler: give up and serve the
// client-side version instead.
const API_TIMEOUT_MS = 5000;

// Where the sitemap is served, relative to the proxy subpath: Shopify owns
// /sitemap.xml at the domain root, and an app proxy only answers below its own prefix.
const SITEMAP_PATH = "/sitemap.xml";

// How long a crawler may reuse the sitemap. Shopify does not CDN-cache app proxy
// responses, so this only reaches the client that asked.
const SITEMAP_MAX_AGE_S = 3600;

// Your page. Liquid is rendered here in the context of the shop's theme, so the parts
// that are not produced by JavaScript can come from Shopify rather than from this file:
// {{ pages['<handle>'].content }} for content edited in the admin, {% render %} for a
// theme snippet, {% section %} for a theme section, plus settings, localization and
// request. Wrap third-party markup in {% raw %} if it contains {{ }} or {% %}.
const PAGE = `<div id="app"></div>
<script src="https://example.com/your-application.js" async></script>`;

export const loader = async ({ request }) => {
    // Verifies that the request was signed by Shopify: the proxy URL is public.
    const { liquid } = await authenticate.public.appProxy(request);

    const url = new URL(request.url);
    const userAgent = request.headers.get("user-agent");

    if (appRelativePath(url) === SITEMAP_PATH) {
        // 503 rather than the page: the client asked for XML, and "come back later" is
        // what an unreadable sitemap actually means.
        return (
            (await fetchSitemap()) ??
            new Response("Sitemap unavailable\n", {
                status: 503,
                headers: { "content-type": "text/plain; charset=utf-8", "retry-after": "300" },
            })
        );
    }

    if (userAgent && !SCRAPER_TEST.test(userAgent) && USER_AGENT_TEST.test(userAgent)) {
        if (!FILENAME_EXTENSION_TEST.test(storefrontPath(url))) {
            const snapshot = await fetchSnapshot(url, request);
            if (snapshot) {
                return snapshot;
            }
        }
    }

    return liquid(PAGE);
};

// What the visitor asked for below the proxy subpath, with the app's own mount point
// removed: "" for /apps/my-page, "/sitemap.xml" for /apps/my-page/sitemap.xml.
function appRelativePath(url) {
    return url.pathname.startsWith(APP_PATH) ? url.pathname.slice(APP_PATH.length) : url.pathname;
}

// A crawler asked for the storefront URL (/apps/my-page/...), not the app's internal
// path, and the storefront URL is the one SEO4Ajax has a snapshot for. Shopify passes
// the storefront prefix along as path_prefix.
function storefrontPath(url) {
    const prefix = url.searchParams.get("path_prefix") ?? APP_PATH;

    return prefix + appRelativePath(url);
}

function storefrontSearch(url) {
    const params = new URLSearchParams(url.search);
    SHOPIFY_PROXY_PARAMS.forEach((param) => params.delete(param));
    const search = params.toString();

    return search ? `?${search}` : "";
}

// Returns null whenever the client-side version should be served instead.
async function fetchSnapshot(url, request) {
    if (!SITE_TOKEN) {
        console.error("[seo4ajax] SEO4AJAX_SITE_TOKEN is not set");
        return null;
    }

    const target = API_URL + SITE_TOKEN + storefrontPath(url) + storefrontSearch(url);

    try {
        const response = await fetch(target, {
            headers: forwardedHeaders(request),
            // Pass a prerendered redirect on to the crawler rather than following it here.
            redirect: "manual",
            signal: AbortSignal.timeout(API_TIMEOUT_MS),
        });

        if (response.status >= 400) {
            console.warn(`[seo4ajax] ${response.status} from ${target}`);
            return null;
        }

        // A response that is not application/liquid is returned to the client
        // untouched, which is what a snapshot needs: it is already a complete
        // document, and any {{ }} in its JSON-LD or inline scripts must not be
        // interpreted as Liquid.
        const headers = new Headers({ "content-type": "text/html; charset=utf-8" });
        const location = response.headers.get("location");
        if (location) {
            headers.set("location", location);
        }

        return new Response(await response.text(), { status: response.status, headers });
    } catch (error) {
        console.warn(`[seo4ajax] ${error.message} while fetching ${target}`);
        return null;
    }
}

// Returns null whenever the sitemap could not be read, so the caller can answer 503. No
// user agent and no query string are passed on: unlike a snapshot, the sitemap is one
// document, identical for every caller.
async function fetchSitemap() {
    if (!SITE_TOKEN) {
        console.error("[seo4ajax] SEO4AJAX_SITE_TOKEN is not set");
        return null;
    }

    const target = API_URL + SITE_TOKEN + SITEMAP_PATH;

    try {
        const response = await fetch(target, { signal: AbortSignal.timeout(API_TIMEOUT_MS) });

        if (!response.ok) {
            console.warn(`[seo4ajax] ${response.status} from ${target}`);
            return null;
        }

        const headers = new Headers({
            "content-type": response.headers.get("content-type") ?? "application/xml",
            "cache-control": `public, max-age=${SITEMAP_MAX_AGE_S}`,
        });
        const robotsTag = response.headers.get("x-robots-tag");
        if (robotsTag) {
            headers.set("x-robots-tag", robotsTag);
        }

        return new Response(await response.text(), { headers });
    } catch (error) {
        console.warn(`[seo4ajax] ${error.message} while fetching ${target}`);
        return null;
    }
}

// Let SEO4Ajax see which crawler this is and where it came from.
function forwardedHeaders(request) {
    const headers = new Headers();
    const userAgent = request.headers.get("user-agent");
    const forwardedFor = request.headers.get("x-forwarded-for");

    if (userAgent) {
        headers.set("user-agent", userAgent);
    }
    if (forwardedFor) {
        headers.set("x-forwarded-for", forwardedFor);
    }

    return headers;
}
```

## Notes

**Query parameters.** Shopify forwards the original query string and appends `shop`,
`path_prefix`, `timestamp`, `signature` and `logged_in_customer_id`. Only the original
parameters belong in the snapshot URL, which is what `storefrontSearch` takes care of.
The browser's address stays `/apps/my-page?...`, so each combination of parameters is a
distinct URL that SEO4Ajax prerenders separately, and client-side code reading
`location.search` behaves as it does anywhere else. Values kept in the fragment
(`#...`) cannot be prerendered, since a fragment never reaches the server.

**Forwarded headers.** Shopify passes the visitor's `user-agent` through to the app,
which is what makes the crawler test possible, along with `x-forwarded-for` and
`x-forwarded-host`.

**The `<head>` of the page.** For a visitor, the theme's layout supplies the `<head>`,
and it has no page context to work from: the title is the shop's name and the canonical
URL excludes query parameters. Client-side applications that set `document.title` or
rewrite the canonical link solve this for crawlers by construction, because the
snapshot is taken after JavaScript has run. To control the `<head>` server-side
instead, render the response without the theme by passing `{ layout: false }` to
`liquid()`, and produce the whole document yourself.

**The sitemap.** SEO4Ajax publishes a sitemap of the URLs it has prerendered, at
`https://api.seo4ajax.com/<site-token>/sitemap.xml`, and `fetchSitemap` re-serves it
under the proxy. It cannot replace the shop's own `/sitemap.xml`: that one is generated
by Shopify, does not list app proxy URLs, and an app proxy cannot answer a root path.
Hence a second sitemap, declared from `robots.txt.liquid`.

Its `<loc>` entries are the URLs of the site as registered in SEO4Ajax, so that site's
domain and paths have to be the storefront's before you add the `Sitemap` directive — on
a live shop, a sitemap pointing at another host is worse than no sitemap at all. The code
above deliberately does not rewrite them: a sitemap has to list URLs that exist, and the
storefront's cannot be derived from another site's by substituting a hostname.

Two smaller points. The response carries SEO4Ajax's `X-Robots-Tag: noindex`, which keeps
the sitemap file itself out of the index without affecting the URLs listed inside it. And
SEO4Ajax returns a single flat `<urlset>`; if a site ever passed the 50,000-URL limit of
the sitemap protocol and had to be split behind a sitemap index, the child sitemaps would
need to be reachable through the proxy too, which the code above does not do.

On a shop with several domains, `Sitemap: {{ request.origin }}/apps/my-page/sitemap.xml`
would make the directive follow whichever host is being served. The `request` object is
not documented as available in `robots.txt.liquid`, so check that it renders before
relying on it, and otherwise write the primary domain out in full.

**Linking.** Parameterised URLs also need ordinary `<a href>` links if crawlers are to
reach them by following the site rather than only through the sitemap.

**During development.** `shopify app dev` publishes a tunnel URL that changes every
time it restarts, and the app proxy URL registered for a shop does not follow it
automatically: after a restart, re-open the app once (press `p` in the dev console) so
the shop points at the current tunnel, otherwise Shopify keeps requesting a tunnel that
no longer exists and the page answers with "There was an error in the third-party
application". Passing a fixed hostname with `--tunnel-url` avoids this entirely.
