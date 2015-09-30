# Minotar Stack Deployment

This represents a fair idea of how the Minotar deployment of imgd should be. We use a combination of caching the Mojang API responses in Redis and caching the processed resource in Varnish.

## Configuration

Check group_vars and production file etc...

Add imgd binary to `roles/backend/files/imgd`

Full deploy:
`ansible-playbook -i production frontendservers.yml`

Deploy Cache:
`ansible-playbook -i production cacheservers.yml`

Redeploy Varnish and Nginx configurations:
`ansible-playbook -i production frontendservers.yml -t configuration`

Redeploy Website:
`ansible-playbook -i production frontendservers.yml -t site`

Redeploy imgd (binary & config):
`ansible-playbook -i production backendservers.yml -t imgd`

## Varnish Breakdown

Varnish is configured with at least 2 backends.
 * Nginx Website
 * imgd Worker(s)

We direct specific requests for certain web resources to the Nginx backend and all other requests are served by a round-robin director which will share load between the imgd workers.

Idea behind the round robin is that we logically want a fairly even distributuion of requests from each of the IPs.

The web resources which will go through to Varnish are as follows:
* /
* /favicon.ico
* /robots.txt
* /404.html
* ^/assets/ (/assets/*)
* ^/addons/ (/addons/*)

Varnish will return a 403 on all connections without a valid (minotar.net or www.minotar.net) Host header. It will also then redirect away from the "www." variant while respecting a X-Forwarded-Proto header if present.

To help ensure we keep a good cache rate we also drop (modify request URL) any "?", "&" and "#" characters as well as all remaining characters until the end of the line. These are not specifically required but would still allow a client to bypass local cache without causing us any strain.

Varnish will drop all cookies and attempts at setting cookies from the backend. This should not interfere with anything like Google Analytics as that is JavaScript based and does not rely on the server accessing the cookies. None of the backends require/should interact with cookies.

We will change any request method that is not GET or HEAD to instead be a GET. This might be overly aggressive, but the aim to neatly prevent anything unexpected getting through to the backends.

There is a "^/stats/$backend" endpoint which has a reduced caching time set (5 seconds, no grace). It forces the backend allowing stats to be taken from specific sources. It modifies the request URL for each backend to also allow caching of them individually, but not as one.

Hashing is based upon Request URL and Host/IP. It could almost certainly be just set to URL and the Host should always be the same. Not sure if there is a observable benefit from this though? Maybe ns from a reduced hash time!?

We will attempt to cache resources without a header set for a generic 2 mins. Will also keep in the cache for a 6 hour grace period which allows the serving of resources even when the backend goes down.

## Other Stuff

Less special and needs some text here
