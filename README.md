# Github Pages

## Todo
- sitemap http://0.0.0.0:4000/sitemap.xml
- rss feed http://0.0.0.0:4000/feed.xml
- index in google
- https://github.com/benbalter/jekyll-relative-links
- https://github.com/jekyll/jekyll-gist
- https://github.com/github/jekyll-commonmark-ghpages

### Later
- https://jekyllrb.com/docs/pagination/

## Run locally

https://github.com/Starefossen/docker-github-pages

```bash
docker run -it --rm -v "$PWD":/usr/src/app -p "4000:4000" starefossen/github-pages jekyll serve -d /_site --watch --force_polling -H 0.0.0.0 -P 4000 --drafts
```

## Dependencies

- https://github.com/github/pages-gem
- https://github.com/jekyll/minima/
- https://cookieconsent.osano.com/download/
- https://fontawesome.com/v4.7.0/icons/

## Docs
- https://help.github.com/en/articles/using-jekyll-as-a-static-site-generator-with-github-pages
- https://help.github.com/en/categories/customizing-github-pages
- https://help.github.com/en/articles/customizing-css-and-html-in-your-jekyll-theme
- https://shopify.github.io/liquid/
- https://pages.github.com/themes/
- https://jekyllrb.com/docs/themes/
- https://pages.github.com/versions/

### Markup
- https://github.github.com/gfm/
- https://www.webfx.com/tools/emoji-cheat-sheet/
