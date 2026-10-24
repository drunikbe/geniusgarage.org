# geniusgarage.org

One static page (`index.html`) that points visitors to Genius Garage Racing.
It is served by `nginx:1-alpine` from the image
`ghcr.io/drunikbe/geniusgarage.org`, on the production Docker Swarm.

## How it deploys

- A push to `main` runs `.github/workflows/deploy.yml`: it builds the image,
  pushes it as `:sha-<commit>`, and commits `deploy/production/compose.yaml`
  with that tag to the `deploy` branch. CI is the only writer of `deploy`.
- Arcane Git Sync applies `deploy/production/compose.yaml` from the `deploy`
  branch as the swarm stack `geniusgarage` (services `web` and `cloudflared`,
  two replicas each, on nodes with `role == apps`).
- The `main` copy of the compose names `:sha-unpinned`, a tag that does not
  exist: it is never deployed as it is.
- Changes go to `main` by pull request.

**GHCR package visibility:** the package `geniusgarage.org` is public, so the
swarm pulls the image without credentials. Keep it public: a private package
makes every pull fail.

## Ingress

- The stack has its own Cloudflare Tunnel, `geniusgarage.org (production)`.
  `cloudflared` dials out; no port is published.
- The tunnel token is the external Swarm secret `geniusgarage_cloudflared`,
  created once on a manager. No secret is in this repo.
- The public hostnames `geniusgarage.org` and `www.geniusgarage.org` point to
  `http://web:80` in the Zero Trust dashboard. A Cloudflare Redirect Rule sends
  `www` to the apex.

## Indexing

The image fails closed: it sends `X-Robots-Tag: noindex, nofollow` and a
`Disallow: /` robots.txt unless `ROBOTS_TAG` is empty. The production compose
sets `ROBOTS_TAG: ''`, so production is indexable.

## Rollback

Revert the pin commit on the `deploy` branch and push:

```sh
git fetch origin deploy
git switch deploy
git revert <pin-commit>
git push origin deploy
```

Git Sync then applies the previous tag. Never run `docker stack deploy` by
hand.

## Test the image locally

```sh
docker build -t geniusgarage:local .
docker run -d --rm -p 8089:80 -e ROBOTS_TAG='' --name gg geniusgarage:local
curl -sI localhost:8089/        # 200, no X-Robots-Tag
docker rm -f gg
```
