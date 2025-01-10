# M1 theme for Jekyll
This theme is expanded version of [Tale theme](https://github.com/chesterhow/tale) by [Chester How](https://github.com/chesterhow)

## Github pages setup
1. Fork or clone this repository
2. Delete the unnecessary files/folders: `CODE_OF_CONDUCT.md`, `LICENSE`, `README.md`, `tale.gemspec`
3. Delete the `baseurl` line in `_config.yml`:

```yaml
baseurl:  "/tale"   # delete this line
```
Once you've installed the theme, you're ready to work on your Jekyll site. To start off, I would recommend updating `_config.yml` with your site's details.

### Enabling Comments
Comments are disabled by default. To enable them, look for the following line in `_config.yml` and change `jekyll-tale` to your site's Disqus id.

```yml
disqus: jekyll-tale
```

Next, add `comments: true` to the YAML front matter of the posts which you would like to enable comments for.

## License
See [LICENSE](https://github.com/M1chol/m1/LICENSE)

## Usage
Look at `post_examples` and move them to `_posts` to see them on page. Base variables
1. title: 
2. author: 

expanded variables:
1. layout:
2. excerpt_separator: 
3. image:
4. tags: Tale
5. comments:
6. sticky:
7. hidden:
