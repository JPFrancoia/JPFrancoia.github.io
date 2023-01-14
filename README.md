## Running locally

source: https://ddewaele.github.io/running-jekyll-in-docker/

```
export JEKYLL_VERSION=3.5
```

Build the site:

```
docker run --rm --volume="$PWD:/srv/jekyll" -it jekyll/jekyll:$JEKYLL_VERSION jekyll build
```

Run/serve the site:

```
docker run --name newblog --volume="$PWD:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll:$JEKYLL_VERSION jekyll serve --watch --drafts
```
