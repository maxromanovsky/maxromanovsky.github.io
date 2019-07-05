# Github Pages

## Todo
- css issue
- tags
- blog listing pagination
- "prod" testing
- Responsive
- Comments
- Google Analytics
- cookie warning
- social links not showing up
- show_excertps
- images
- sitemap http://0.0.0.0:4000/sitemap.xml
- rss feed http://0.0.0.0:4000/feed.xml

## Run locally

https://github.com/Starefossen/docker-github-pages

```bash
docker run -it --rm -v "$PWD":/usr/src/app -p "4000:4000" starefossen/github-pages
```

## Dependencies

- https://github.com/github/pages-gem
- https://github.com/jekyll/minima
- https://cookieconsent.osano.com/download/
