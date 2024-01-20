---
title: Using the URL Rewrite middleware with Tyk OAS APIs
date: 2024-01-17
description: "Using the URL Rewrite middleware with Tyk OAS APIs"
tags: ["URL rewrite", "middleware", "per-endpoint", "Tyk OAS"]
---

## Overview
Tyk's [URL rewriter]({{< ref "transform-traffic/url-rewriting" >}}) uses the concepts of triggers and rules to determine if the request (target) URL should be modified. These can be combined in flexible ways to create sophisticated logic to direct requests made to a single endpoint to various upstream services (or other APIs internally exposed within Tyk).

URL rewrite triggers and rules are explained in detail [here]({{< ref "product-stack/tyk-gateway/middleware/url-rewrite-middleware" >}}).

When working with Tyk OAS APIs the rules and triggers are configured in the [Tyk OAS API Definition]({{< ref "tyk-apis/tyk-gateway-api/oas/x-tyk-oas-doc#operation" >}}); this can be done manually within the `.json` file or from the API Designer in the Tyk Dashboard.

If you're using the legacy Tyk Classic APIs, then check out the [Tyk Classic]({{< ref "/product-stack/tyk-gateway/middleware/url-rewrite-tyk-classic" >}}) page.

## Configuring the URL rewriter in the Tyk OAS API Definition

The URL rewrite middleware (`urlRewrite`) can be added to the `operations` section of the Tyk OAS Extension (`x-tyk-api-gateway`) in your Tyk OAS API Definition for the appropriate `operationId` (as configured in the `paths` section of your OpenAPI Document).

For the basic trigger, you only need to enable the middleware (set `enabled:true`) and then configure the `pattern` and the `rewriteTo` (target) URL. The design of the Tyk OAS API Definition takes advantage of the `operationID` [defined in the OpenAPI Document]({{< ref "getting-started/key-concepts/paths#operation-id" >}}) that declares both the path and method required by the basic trigger.

```{.json}
{
    "openapi": ...,
    "paths": {
        "/books/author": {
            "get": {
                "operationId": "books/authorget",
                ...
            }
        }
    }
}
...
"x-tyk-api-gateway": {
    ...
    "middleware": {
        "operations": {
            "books/authorget": {
                "urlRewrite": {
                    "enabled": true,
                    "pattern": "(\w+)/(\w+)",
                    "rewriteTo": "library/service?value1=$1&value2=$2",
                }
            }
        }
    }
}
```

In this example the basic trigger has been configured to match the path for an HTTP `GET` request to the `/books/author` endpoint against the pure regex `(\w+)/(\w+)`. This is looking for two word groups in the path which, if found, will store the first string (`books`) in context variable `$1` and the second (`author`) in `$2`. The request (target) URL will then be rewritten to `library/service?value1=books&value2=author` ready for processing by the next middleware in the chain.

You can add advanced triggers to your URL rewriter configuration by adding the `triggers` element within the `urlRewrite` middleware configuration for the operation.

The `triggers` element is an array, with one entry per advanced trigger. For each of those triggers you configure:
 - `condition` to set the logical condition to be applied to the rules (`any` or `all`)
 - `rules` a list of rules for the trigger
 - `rewriteTo` the address to which the (target) URL should be rewritten if the trigger fires

The rules are defined using this format:
```
{
    "in": key_location,
    "name": key_name,
    "pattern": pattern,
    "negate": true/false (set to true to trigger if pattern does not match)
}
```

Key locations are encoded as follows:
 - `header` - request header parameter
 - `query` - query parameter
 - `path` - path parameter (i.e. components of the path itself)
 - `sessionMetadata` - session metadata
 - `requestBody`- request body
 - `requestContext`- request context

For example:

```{.json}
{
    "openapi": ...,
    "paths": {
        "/books/author": {
            "get": {
                "operationId": "books/authorget",
                ...
            }
        }
    }
}
...
"x-tyk-api-gateway": {
    ...
    "middleware": {
        "operations": {
            "books/authorget": {
                "urlRewrite": {
                    "enabled": true,
                    "pattern": "(\w+)/(\w+)",
                    "rewriteTo": "library/service?value1=$1&value2=$2",
                    "triggers": [
                        {
                            "condition": any,
                            "rules": [
                                {
                                    "in": "query",
                                    "name": "genre",
                                    "pattern": "fiction",
                                    "negate": "false",
                                }
                            ]
                            "rewriteTo": "library/service/author?genre=$tyk_context.trigger-0-genre-0", 
                        }
                        {
                            "condition": all,
                            "rules": [
                                {
                                    "in": "header",
                                    "name": "X-Enable-Beta",
                                    "pattern": "true",
                                    "negate": "false",
                                }
                                {
                                    "in": "sessionData",
                                    "name": "beta_enabled",
                                    "pattern": "true",
                                    "negate": "false",
                                }
                            ]
                            "rewriteTo": "https://beta.library.com/books/author",    
                        }
                    ]
                }
            }
        }
    }
}
```
In this example, the basic trigger is configured as before, but two advanced triggers have been added.

The first advanced trigger has this configuration:
 - key location is query parameter
 - key name is genre
 - pattern is fiction

So if a `GET` request is made to `/books/author?genre=fiction` the advanced trigger will fire and the URL will be rewritten to `library/service/author?genre=fiction`.

The second advanced trigger has this configuration:
 - rule condition: ALL
 - rule 1
    - key location is header parameter
    - key name is `X-Enable-Beta`
    - pattern is `true``
 - rule 2
    - key location is session metadata
    - key name is `beta_enabled`
    - pattern is `true`

So if a `GET` request is made to `/books/author` with a header `"X-Enable-Beta":"true"` and, within the session metadata, `"beta_enabled":"true"` the second advanced trigger will fire and the URL will be written to `https://beta.library.com/books/author` taking the request to a different upstream host entirely.

## Configuring the URL rewriter in the API Designer

Adding and configuring the URL rewrite feature to your API endpoints is easy when using the API Designer in the Tyk Dashboard, simply follow the steps taken in this short video:

< placeholder for video >
