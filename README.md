# Github Pages

## Todo

### Later
- https://github.com/benbalter/jekyll-relative-links
- https://jekyllrb.com/docs/pagination/
- https://github.com/jekyll/jekyll-gist

## Run locally

https://github.com/Starefossen/docker-github-pages

```bash
# in other dir
git clone https://github.com/github/pages-gem.git
cd pages-gem
make image
# in current dir
docker run -it --rm -v "$PWD":/src/site -p "4000:4000" gh-pages /bin/bash -c "bundle install --gemfile=/src/site/Gemfile; jekyll serve --watch --force_polling -H 0.0.0.0 -P 4000 --drafts"
```

## Dependencies

- https://pages.github.com/versions/
- https://github.com/github/pages-gem
- https://github.com/jekyll/minima/
- https://cookieconsent.osano.com/download/
- https://fontawesome.com/v4.7.0/icons/
    - https://www.bootstrapcdn.com/fontawesome/

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
- https://guides.github.com/features/mastering-markdown/
- https://www.webfx.com/tools/emoji-cheat-sheet/
- https://github.com/ikatyang/emoji-cheat-sheet/blob/master/README.md
