About
=====
`ngx_cache_purge` is `nginx` module which adds ability to purge content from
`FastCGI`, `proxy`, `SCGI` and `uWSGI` caches. A purge operation removes the 
content with the same cache key as the purge request has.


Sponsors
========
Work on the original patch was fully funded by [yo.se](http://yo.se).


Status
======
This module is production-ready.


Configuration directives (same location syntax)
===============================================
fastcgi_cache_purge
-------------------
* **syntax**: `fastcgi_cache_purge on|off|<method> [purge_all] [from all|<ip> [.. <ip>]]`
* **default**: `none`
* **context**: `http`, `server`, `location`

Allow purging of selected pages from `FastCGI`'s cache.


proxy_cache_purge
-----------------
* **syntax**: `proxy_cache_purge on|off|<method> [purge_all] [from all|<ip> [.. <ip>]]`
* **default**: `none`
* **context**: `http`, `server`, `location`

Allow purging of selected pages from `proxy`'s cache.


scgi_cache_purge
----------------
* **syntax**: `scgi_cache_purge on|off|<method> [purge_all] [from all|<ip> [.. <ip>]]`
* **default**: `none`
* **context**: `http`, `server`, `location`

Allow purging of selected pages from `SCGI`'s cache.


uwsgi_cache_purge
-----------------
* **syntax**: `uwsgi_cache_purge on|off|<method> [purge_all] [from all|<ip> [.. <ip>]]`
* **default**: `none`
* **context**: `http`, `server`, `location`

Allow purging of selected pages from `uWSGI`'s cache.


Configuration directives (separate location syntax)
===================================================
fastcgi_cache_purge
-------------------
* **syntax**: `fastcgi_cache_purge zone_name key`
* **default**: `none`
* **context**: `location`

Sets area and key used for purging selected pages from `FastCGI`'s cache.


proxy_cache_purge
-----------------
* **syntax**: `proxy_cache_purge zone_name key`
* **default**: `none`
* **context**: `location`

Sets area and key used for purging selected pages from `proxy`'s cache.


scgi_cache_purge
----------------
* **syntax**: `scgi_cache_purge zone_name key`
* **default**: `none`
* **context**: `location`

Sets area and key used for purging selected pages from `SCGI`'s cache.


uwsgi_cache_purge
-----------------
* **syntax**: `uwsgi_cache_purge zone_name key`
* **default**: `none`
* **context**: `location`

Sets area and key used for purging selected pages from `uWSGI`'s cache.

Configuration directives (Optional)
===================================================

cache_purge_response_type
-----------------
* **syntax**: `cache_purge_response_type html|json|xml|text`
* **default**: `html`
* **context**: `http`, `server`, `location`

Sets a response type of purging result.



Partial Keys
============
Sometimes it's not possible to pass the exact key cache to purge a page. For example; when the content of a cookie or the params are part of the key.
You can specify a partial key adding an asterisk at the end of the URL.

    curl -X PURGE /page*

The asterisk must be the last character of the key, so the cache key must end with the URI or request URI portion.



Refresh (conditional cache validation)
======================================
Instead of blindly purging all matched cache entries, `refresh` sends conditional
validation subrequests upstream using `If-None-Match` / `If-Modified-Since`
headers extracted from each cached file. Nginx may still translate the upstream
method internally, but refresh keeps the request header-only and does not read the
response body. Only entries that actually changed (upstream returns `200`) are
purged; unchanged entries (upstream returns `304`) are kept.

Refresh is **proxy_cache only**. `proxy_cache_refresh` is the dedicated refresh
directive, while exact/partial requests can also reach refresh semantics through a
`proxy_cache_purge` entrypoint when the HTTP method is `REFRESH`.
Refresh is not supported for `fastcgi`, `scgi`, or `uwsgi` caches.

### Directives

proxy_cache_refresh
-------------------
* **syntax**:
  - same location: `proxy_cache_refresh <method> [refresh_all] [from all|<ip> [.. <ip>]]`
  - separate location: `proxy_cache_refresh <zone> <key>`
* **default**: `none`
* **context**: `http`, `server`, `location`

Enable refresh mode with a dedicated directive. In same-location form, the first
argument declares the location's configured entry method (for example `REFRESH`).
Optional `refresh_all` enables full-zone refresh capability for that location.
In separate-location form, the directive configures the cache zone and cache key
directly on the control endpoint.

cache_purge_refresh_timeout
---------------------------
* **syntax**: `cache_purge_refresh_timeout <time>`
* **default**: `300s`
* **context**: `http`, `server`, `location`

Maximum wall-clock time for the entire refresh operation. When exceeded, no new
subrequests are dispatched; already in-flight subrequests complete naturally.
Remaining unprocessed entries are counted as errors.

cache_purge_refresh_concurrency
-------------------------------
* **syntax**: `cache_purge_refresh_concurrency <number>`
* **default**: `32`
* **context**: `http`, `server`, `location`

Maximum number of concurrent validator subrequests during a refresh operation.

### Important Rules

- One `location` can configure only one entry directive: `proxy_cache_purge` or `proxy_cache_refresh`.
- For exact and partial requests, both entry directives accept both `PURGE` and `REFRESH`.
- For exact and partial requests, the actual action is decided by the HTTP method, not by the directive name.
- Successful purge and refresh responses include `X-Cache-Action: purge` or `X-Cache-Action: refresh`.
- Full-zone requests are not freely interchangeable: `PURGE` needs `purge_all`, and `REFRESH` needs `refresh_all`.
- A full-zone capability mismatch returns `400 Bad Request`.
- `purge_all` belongs to `proxy_cache_purge`; `refresh_all` belongs to `proxy_cache_refresh`.
- `proxy_cache_refresh ... purge_all ...` is invalid configuration and is rejected at config load time.
- Refresh is `proxy_cache` only. It is not available for `fastcgi_cache`, `scgi_cache`, or `uwsgi_cache`.
- Any proxy location that participates in the refresh request path must include both:

      proxy_cache_bypass  $cache_purge_refresh_bypass;
      proxy_no_cache      $cache_purge_refresh_bypass;

- The cache key must end with the URI or request URI portion. It is not limited to `$uri$is_args$args`, but the URI/request URI must be at the tail of the key.

### Method Routing Model

For exact and partial requests, runtime routing is method-driven:

| Configured entry directive | Request method | Actual action |
| --- | --- | --- |
| `proxy_cache_purge` | `PURGE` | purge |
| `proxy_cache_purge` | `REFRESH` | refresh |
| `proxy_cache_refresh` | `REFRESH` | refresh |
| `proxy_cache_refresh` | `PURGE` | purge |

This table applies only to exact and partial requests. Full-zone requests use the
capability rules in the next section.

### Capability Matrix

For full-zone requests, the request method and the configured full capability must match:

| Full-zone request | Required configured capability | Failure result |
| --- | --- | --- |
| `PURGE /.../*` | `purge_all` on `proxy_cache_purge` | `400 Bad Request` |
| `REFRESH /.../*` | `refresh_all` on `proxy_cache_refresh` | `400 Bad Request` |

Configuration-time rules are also strict:

- `proxy_cache_purge ... purge_all ...` is valid.
- `proxy_cache_refresh ... refresh_all ...` is valid.
- `proxy_cache_refresh ... purge_all ...` is invalid.
- `proxy_cache_purge` and `proxy_cache_refresh` in the same `location` are invalid.

### Recommended Configurations

Recommended production layout: expose separate purge and refresh endpoints.

    http {
        proxy_cache_path  /tmp/cache  keys_zone=tmpcache:10m;

        server {
            location / {
                proxy_pass         http://127.0.0.1:8000;
                proxy_cache        tmpcache;
                proxy_cache_key    "$scheme$proxy_host$request_uri";
                proxy_cache_bypass $cache_purge_refresh_bypass;
                proxy_no_cache     $cache_purge_refresh_bypass;
            }

            location ~ /purge(/.*) {
                allow              127.0.0.1;
                deny               all;
                proxy_cache_purge  tmpcache $scheme$proxy_host$1$is_args$args;
            }

            location ~ /refresh(/.*) {
                allow              127.0.0.1;
                deny               all;
                proxy_pass         http://127.0.0.1:8000;
                proxy_cache_bypass $cache_purge_refresh_bypass;
                proxy_no_cache     $cache_purge_refresh_bypass;
                proxy_cache_refresh            tmpcache $scheme$proxy_host$1$is_args$args;
                cache_purge_refresh_timeout     60s;
                cache_purge_refresh_concurrency 32;
            }
        }
    }

Gradual migration layout: keep one entry directive and migrate clients by switching
HTTP method first. Exact and partial requests already route by method.

    http {
        proxy_cache_path  /tmp/cache  keys_zone=tmpcache:10m;

        server {
            location / {
                proxy_pass         http://127.0.0.1:8000;
                proxy_cache        tmpcache;
                proxy_cache_key    "$host$request_uri";
                proxy_cache_bypass $cache_purge_refresh_bypass;
                proxy_no_cache     $cache_purge_refresh_bypass;
                proxy_cache_purge  PURGE from 127.0.0.1;
            }
        }
    }

In that layout:

- `PURGE /path/file` performs purge.
- `REFRESH /path/file` performs refresh.
- Full-zone `REFRESH` still needs a dedicated `proxy_cache_refresh ... refresh_all ...` location.

This migration trick does not remove refresh prerequisites. If a request is routed
to refresh, the proxy path participating in that refresh flow still needs the
refresh bypass rules, and the cache key still needs to end with the URI or
request URI portion.

### Common Misconfigurations

- Do not put both entry directives in one `location`:

      location /control/ {
          proxy_cache_purge   PURGE from 127.0.0.1;
          proxy_cache_refresh REFRESH from 127.0.0.1;
      }

- Do not expect full-zone `PURGE` to work on a `refresh_all` location, or full-zone `REFRESH` to work on a `purge_all` location. Those fail with `400 Bad Request`.

- Do not write `proxy_cache_refresh REFRESH purge_all from 127.0.0.1;`. Use `refresh_all`, not `purge_all`.

- Do not omit refresh bypass rules from proxy locations used by refresh:

      proxy_cache_bypass $cache_purge_refresh_bypass;
      proxy_no_cache     $cache_purge_refresh_bypass;

- Do not use a cache key where the URI appears in the middle, such as `$arg_x$uri$host`. The URI or request URI must be at the end of the key.

- Do not try refresh on `fastcgi`, `scgi`, or `uwsgi` caches.

### Response Format

Refresh success responses return `200 OK`. The exact body format depends on
`cache_purge_response_type`, but refresh currently supports only JSON or text
output. If `cache_purge_response_type` is `json`, refresh returns JSON; all other
values fall back to text. In the default text format, the body looks like:

    Refresh: total=<N> kept=<K> purged=<P> errors=<E>

Where:
- `total`: number of matched cache entries scanned
- `kept`: entries where upstream returned `304` or a race kept the cache entry
- `purged`: entries invalidated after upstream returned `200`, `404`, or `410`
- `errors`: entries that failed conservatively (subrequest error, timeout, upstream `403`/`500`, and so on)

Successful purge and refresh responses also include:

    X-Cache-Action: purge

or:

    X-Cache-Action: refresh

### How it works

Refresh subrequests use conditional validator headers (`If-None-Match` /
`If-Modified-Since`) so unchanged objects can be kept with `304 Not Modified`
responses. Internally nginx may still convert the upstream request method to
`GET`, but refresh forces header-only handling and does not read the response
body. The bandwidth savings therefore come from validator-based `304` responses
and from avoiding response-body reads on the refresh path.

Subrequests use nginx's background subrequest mechanism (`NGX_HTTP_SUBREQUEST_BACKGROUND`)
to avoid `r->main->count` overflow. This allows refresh to handle 100,000+ cached
entries in a single request without hitting nginx's 64535 subrequest limit.

Current upstream status policy during refresh is intentionally conservative:

- `304 Not Modified`: keep the cache entry
- `200 OK`: treat as changed and run the normal invalidate path
- `404 Not Found` / `410 Gone`: purge the cache entry because the upstream object is gone
- other statuses (for example `403`, `429`, `500`) and subrequest failures: keep the cache entry and count an error

At the end of each refresh request, the module also emits one `error_log info`
summary line like:

    cache refresh summary uri="/path/*" total=<N> kept=<K> purged=<P> errors=<E> timed_out=<0|1>

Per-entry decisions remain debug-level logs.

Usage:

    # Refresh a single file (only purge if changed)
    curl -X REFRESH http://localhost/refresh/path/to/file.txt

    # Refresh all files under /images/ (wildcard)
    curl -X REFRESH http://localhost/refresh/images/*

    # Refresh entire cache zone (requires refresh_all)
    curl -X REFRESH http://localhost/refresh/*


Sample configuration (same location syntax - purge only)
========================================================
    http {
        proxy_cache_path  /tmp/cache  keys_zone=tmpcache:10m;

        server {
            location / {
                proxy_pass         http://127.0.0.1:8000;
                proxy_cache        tmpcache;
                proxy_cache_key    "$uri$is_args$args";
                proxy_cache_purge  PURGE from 127.0.0.1;
            }
        }
    }


Sample configuration (same location syntax - refresh only)
==========================================================
    http {
        proxy_cache_path  /tmp/cache  keys_zone=tmpcache:10m;

        server {
            location / {
                proxy_pass           http://127.0.0.1:8000;
                proxy_cache          tmpcache;
                proxy_cache_key      "$uri$is_args$args";
                proxy_cache_bypass   $cache_purge_refresh_bypass;
                proxy_no_cache       $cache_purge_refresh_bypass;
                proxy_cache_refresh  REFRESH from 127.0.0.1;
            }
        }
    }


Sample configuration (same location syntax - purge all cached files)
====================================================================
    http {
        proxy_cache_path  /tmp/cache  keys_zone=tmpcache:10m;

        server {
            location / {
                proxy_pass         http://127.0.0.1:8000;
                proxy_cache        tmpcache;
                proxy_cache_key    "$uri$is_args$args";
                proxy_cache_purge  PURGE purge_all from 127.0.0.1 192.168.0.0/8;
            }
        }
    }


Sample configuration (separate location syntax)
===============================================
    http {
        proxy_cache_path  /tmp/cache  keys_zone=tmpcache:10m;

        server {
            location / {
                proxy_pass         http://127.0.0.1:8000;
                proxy_cache        tmpcache;
                proxy_cache_key    "$uri$is_args$args";
            }

            location ~ /purge(/.*) {
                allow              127.0.0.1;
                deny               all;
                proxy_cache_purge  tmpcache $1$is_args$args;
            }
        }
    }

Sample configuration (Optional)
===============================================
    http {
        proxy_cache_path  /tmp/cache  keys_zone=tmpcache:10m;

        cache_purge_response_type text;

        server {

            cache_purge_response_type json;

            location / { #json
                proxy_pass         http://127.0.0.1:8000;
                proxy_cache        tmpcache;
                proxy_cache_key    "$uri$is_args$args";
            }

            location ~ /purge(/.*) { #xml
                allow              127.0.0.1;
                deny               all;
                proxy_cache        tmpcache;
                proxy_cache_key    "$1$is_args$args";
                cache_purge_response_type xml;
            }

            location ~ /purge2(/.*) { # json
                allow              127.0.0.1;
                deny               all;
                proxy_cache        tmpcache;
                proxy_cache_key    "$1$is_args$args";
            }
        }

        server {

            location / { #text
                proxy_pass         http://127.0.0.1:8000;
                proxy_cache        tmpcache;
                proxy_cache_key    "$uri$is_args$args";
            }

            location ~ /purge(/.*) { #text
                allow              127.0.0.1;
                deny               all;
                proxy_cache        tmpcache;
                proxy_cache_key    "$1$is_args$args";
            }

            location ~ /purge2(/.*) { #html
                allow              127.0.0.1;
                deny               all;
                proxy_cache        tmpcache;
                proxy_cache_key    "$1$is_args$args";
                cache_purge_response_type html;
            }
        }
    }



Solve problems
==============
* Enabling [`gzip_vary`](https://nginx.org/r/gzip_vary) can lead to different results when clearing, when enabling it, you may have problems clearing the cache. For reliable operation, you can disable [`gzip_vary`](https://nginx.org/r/gzip_vary) inside the location [#20](https://github.com/nginx-modules/ngx_cache_purge/issues/20).


Testing
=======
`ngx_cache_purge` comes with complete test suite based on [Test::Nginx](http://github.com/agentzh/test-nginx).

You can test it by running:

`$ prove`


License
=======
    Copyright (c) 2009-2014, FRiCKLE <info@frickle.com>
    Copyright (c) 2009-2014, Piotr Sikora <piotr.sikora@frickle.com>
    All rights reserved.

    This project was fully funded by yo.se.

    Redistribution and use in source and binary forms, with or without
    modification, are permitted provided that the following conditions
    are met:
    1. Redistributions of source code must retain the above copyright
       notice, this list of conditions and the following disclaimer.
    2. Redistributions in binary form must reproduce the above copyright
       notice, this list of conditions and the following disclaimer in the
       documentation and/or other materials provided with the distribution.

    THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
    "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
    LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
    A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
    HOLDERS OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
    SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
    LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
    DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
    THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
    (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
    OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.


See also
========
- [ngx_slowfs_cache](http://github.com/FRiCKLE/ngx_slowfs_cache).
- http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#purger
- http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_cache_purge
- https://github.com/wandenberg/nginx-selective-cache-purge-module
- https://github.com/wandenberg/nginx-sorted-querystring-module
- https://github.com/ledgetech/ledge
- [Faking Surrogate Cache-Keys for Nginx Plus](https://www.innoq.com/en/blog/faking-surrogate-cache-keys-for-nginx-plus/) ([gist](https://gist.github.com/titpetric/2f142e89eaa0f36ba4e4383b16d61474))
- [Delete NGINX cached md5 items with a PURGE with wildcard support](https://gist.github.com/nosun/0cfb58d3164f829e2f027fd37b338ede)
