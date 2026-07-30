# nginx-le
 Configuration nginx as reverse proxy with Let's Encrypt in docker container

How to use:

install docker and docker-compose

docker create network nginx-reverse-proxy

docker-compose pull

docker-compose up -d

if need add bot or anoher service: just add in path etc/bots file "some_bot.conf" as like example lyt_poster and run docker-compose exec nginx nginx -s reload

## Let's Encrypt / ACME challenge

certbot runs inside the container as `certbot certonly --webroot -w /usr/share/nginx/html -d $LE_FQDN`,
so the challenge files exist **only on this host**, in `/usr/share/nginx/html/.well-known/acme-challenge/`.

Two rules follow from that:

1. `/.well-known/acme-challenge/` must never be proxied to a machine that is not the one
   requesting the certificate — it is served locally from `/usr/share/nginx/html`
   (see `etc/acme.conf`).
2. **Every** `server` block that listens on port 80 must contain `include ./acme.conf;`.
   Without it the vhost's own `location /` (a `proxy_pass` to the app) answers the
   challenge with the app's page and certbot fails with `unauthorized`.
   The entrypoint only hides `conf.d` during the very first certificate request,
   so all later renewals (every 10 days) run with the vhosts enabled.

Every domain listed in `LE_FQDN` must resolve to this host and answer the challenge,
otherwise the whole certificate request fails — including the domains that are fine.

Check a domain before/after a renewal:

    docker-compose exec nginx sh -c 'echo ok > /usr/share/nginx/html/.well-known/acme-challenge/test'
    curl http://your.domain/.well-known/acme-challenge/test   # must print "ok"
    docker-compose exec nginx rm /usr/share/nginx/html/.well-known/acme-challenge/test

Apply config changes and request the certificate again:

    docker-compose up -d           # recreate, needed after changing docker-compose.yml
    docker-compose exec nginx nginx -t && docker-compose exec nginx nginx -s reload
    docker-compose restart nginx   # restart re-runs the letsencrypt updater

## Front proxy + origin (two nginx-le instances)

`browser --https--> front proxy --https--> origin`, TLS terminated on both hops. Two
machines each run nginx-le, and each issues certificates only for the domains that
resolve to it:

| domain                     | DNS points at | TLS terminated by | certificate issued by |
|----------------------------|---------------|-------------------|-----------------------|
| `hestia.bomzheg.dev`       | front proxy   | front proxy       | front proxy           |
| `shvatka.bomzheg.dev`      | front proxy   | front proxy       | front proxy           |
| `shvatka-test.bomzheg.dev` | front proxy   | front proxy       | front proxy           |
| `nemesis.bomzheg.dev`      | CDN           | origin            | origin                |

**The origin's address is never published.** No DNS record points at it — not even a
DNS-only one, which would hand the address to anyone who asks. The front proxy reaches it
through `nemesis.internal`, an `extra_hosts` entry in the gitignored
`docker-compose.override.yml`, so the address exists on that one machine and nowhere else.
Hiding it is only real if the origin also refuses connections from everyone else: allow
`:80`/`:443` from the front proxy and from the CDN's ranges (`nemesis.bomzheg.dev` is
validated and served through them), and drop the rest. An origin that answers the whole
internet is found by scanning regardless of DNS.

The hop to the origin is still verified TLS, without any self-signed certificate: nginx
connects to the unlisted address but checks the certificate against
`nemesis.bomzheg.dev` (`proxy_ssl_name`), which Let's Encrypt already issued there.

The rule the table follows from: **the certificate must live on the machine that
terminates TLS for the domain**, because the origin's certificate never reaches the
browser through a terminating proxy. A domain proxied to the origin is still terminated
here, so it is still issued here — the split runs along DNS, not along where the
application happens to run.

Consequences:

1. `LE_FQDN` on each machine lists **only** its own domains. The lists must not overlap:
   one certificate request covers every domain in `LE_FQDN`, so a single unreachable
   domain fails the whole request.
2. Every server block includes `./acme.conf` and answers its own challenge locally.
   **Nothing forwards `/.well-known/acme-challenge/` to the other machine** — a forwarded
   challenge can only ever be answered by tokens the *other* machine's certbot wrote.
3. The internal name is never in `LE_FQDN`: it does not resolve publicly, so Let's Encrypt
   could not validate it and the whole request would fail. Only the origin's real public
   name is issued there, and it is what `proxy_ssl_name` presents and `proxy_ssl_verify`
   checks — nginx does **not** verify upstream certificates unless asked.
4. Roles differ by one mounted file: the origin uses `etc/services.conf`, the front proxy
   mounts `etc/services-hestia.conf` in its place through `docker-compose.override.yml`
   (gitignored, compose merges it automatically — see
   `docker-compose.override.yml.example`). Keep per-machine differences there rather than
   as uncommitted edits to tracked files.

Shrinking a machine's `LE_FQDN` is not free-standing: the certificate it currently holds
may still cover domains that have since moved to the other machine, and one request covers
the whole list. Narrow `LE_FQDN` on the machine that is losing a domain **before** the
other machine stops answering that domain's challenge, or the next renewal fails for the
whole list — including the names that did not move.

If a public name sits behind a CDN that forces HTTPS, disable that redirect for
`*/.well-known/acme-challenge/*`. Recreating the container runs the first certificate
request while the entrypoint still has `conf.d` hidden, so nothing listens on 443 yet and
a redirected challenge lands in a 502. Renewals from the running loop are unaffected —
they happen with the vhosts enabled.

forked from https://github.com/nginx-le/nginx-le
