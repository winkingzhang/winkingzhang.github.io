# Wenqing's essays

Wenqing's essays with GitBook look!


## How to Get Started (as a template)

[Fork][1] this repository and add your markdown posts to the `_posts` folder.

### Deploy Locally with Jekyll Serve

This theme can be run locally using Ruby and Gemfiles.

[Testing your GitHub Pages site locally with Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/testing-your-github-pages-site-locally-with-jekyll) - GitHub

## How to generate TOC

The jekyll-gitbook theme leverages [jekyll-toc][2] to generate the *Contents* for the page.
The TOC feature is not enabled by default. To use the TOC feature, modify the TOC
configuration in `_config.yml`:

```yaml
toc:
    enabled: true
```

## License

This repo is open sourced under the Apache License, Version 2.0.


[1]: https://github.com/winkingzhang/winkingzhang.github.io/fork
[2]: https://github.com/allejo/jekyll-toc
