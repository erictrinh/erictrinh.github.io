---
layout: post
title:  "Tips for using Jekyll with Github Pages"
---

I was recently toying around with the blog and committed a few changes when disaster struck. **Page build failure**. Here's the cryptic error:

```
The page build failed with the following error:

The file `_posts/2013-09-21-hello-world.md/#excerpt` contains syntax errors. For more information, see https://help.github.com/articles/page-build-failed-markdown-errors.
```

Problem was, everything worked perfectly on my local machine. Little did I know that I would be debugging this issue for the next few hours. 

The issue was that my local config file was different from what Github Pages uses, so it was near impossible to figure out what was wrong until I modified mine to mirror Github's. Here are some tips to get as close as you can to the Github Pages build server.

These can be found on the [documentation for using Jekyll with Github Pages](https://help.github.com/articles/using-jekyll-with-pages#installing-jekyll), but I thought I'd distill it to the actionable items, for people who have already set up Jekyll, but never bothered to customize it for Github Pages specifically.

**STEP 1**: Install and run Jekyll through the Github Pages gem, via Bundler. This will ensure that you're running the version of Jekyll that matches what Github Pages uses[^jekyll2]. Create a new file in your Jekyll root directory called `Gemfile` and put this in it:

```
source 'https://rubygems.org'
gem 'github-pages'
```

Now, run `bundle install`. This will install all of Github Pages' dependencies, including Jekyll. Then `bundle exec jekyll serve` to run Jekyll through Bundler.

**STEP 2**: Add these two lines to your `_config.yml`:

```
safe: true
lsi: false
```

These are hard-coded settings that Github Pages will apply to your project, overriding your own settings if they contradict.

**STEP 3**: Rejoice in getting one step closer to the Github Pages production servers.

In my case, applying `safe: true` broke my local server, because I was using a plugin that wasn't supported by Github Pages[^plugins]. I removed the offending plugin and am now as happy as a clam.



[^jekyll2]: Version mismatches are probably more common than you think. Github Pages took a few months to support Jekyll 2.0, for example.

[^plugins]: Github Pages supports a surprisingly small number of plugins. [Here are some](https://help.github.com/articles/using-jekyll-plugins-with-github-pages)
