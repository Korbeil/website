---
layout: post
title:  "From nothing to `baptiste-leduc.now.sh`"
date:   2018-05-08 20:57:55 +0200
categories: travis-ci github open-source zeit jekyll
---

# 1. Introduction

I always wanted to have a clean website, but after iterations, never got satisfied.
Last iteration was based on [Silex](https://silex.symfony.com/) with [Twig](https://twig.symfony.com/) templates.
I also used [Materialize](https://materializecss.com/), a bootstrap brother based on material design.

When I did it, the main feature was a dynamic timeline to show my experiences and other stuff.

Some preview of what it was:
![Last website iteration]({{ "/assets/from-nothing-to-baptiste-leduc-now-sh-1.png" | prepend: site.baseurl }})
If you want to see more about this website, you can check the repository here: [Korbeil/portfolio](https://github.com/Korbeil/portfolio)

But hey, this website is old, I didn't like the code architecture and this style started to be odd.
So, I recently changed my job and decided to make it looks better !

# 2. Symfony

Since this new job was asking me to make a Symfony bundle, this was a great experience and was an opportunity to make this website great again !
So I started with the same design but using the symfony framework.

So all assets where the same, with a cleaner architecture.
Still using [Twig](https://twig.symfony.com/) so all templates where imported really easily.

But on this time, my main purpose wasn't only Symfony, but also making a blog !
So I started playing the great [EasyAdmin bundle](https://symfony.com/doc/master/bundles/EasyAdminBundle/index.html) to get articles and to manage my old content (This "old content" was managed in yaml files on the last iteration).
This was a great experience, and the EasyAdmin is such a big help if you wanna manage entities with ease !

But, this design was still not something I was ready to keep, and I wanted something new, and that don't need me to work on code if I need features.

# 3. Jekyll

So I started to like, forget this website, since the prototype I made wasn't really what I needed.
Started this new job, and some time after, we plan to make some documentation so I started looking on what we can do.

And I discovered [Jekyll](https://jekyllrb.com/) !
Basically, it's is a blog-aware, static site generator in Ruby.

Nothing new here, but what was interesting is that is blog-aware, which was my main focus !

After browsing a lot of [Jekyll themes](http://themes.jekyllrc.org/) I choosed two of them:
- So simple: [mmistakes/so-simple-theme](https://github.com/mmistakes/so-simple-theme)
- The plain: [heiswayi/the-plain](https://github.com/heiswayi/the-plain)

Both seduced me by their simplicity and their look centralized on the blog ;)

After some tests I choosed "The plain" theme !

# 4. Github Pages

But here comes the problems ...

For hosting this website, I choosed Github pages which fully [supports Jekyll](https://help.github.com/articles/using-jekyll-as-a-static-site-generator-with-github-pages/) !

- Basically everytime I used Github Pages, my website wasn't has it was in local environment (like, never).
- URL linking was working weird, like sometimes URL doesn't match a post when it was working with local.
- And worst of all, Github Pages use 302 HTTP responses from time to time to avoid DDoS attacks, which makes it impossible to crawl by usual bots.

I was even thinking on hosting that simple static website on a server ...
But for development purpose, I continued using Github Pages while looking for something else to save me.

During that time I overwrited a lot of core module layouts / included files to make the website more personal.
You can find all this modifications there: [Korbeil/website](https://github.com/Korbeil/website)

# 5. Zeit & now

And then, I found Zeit !

This wasn't really planned to find some cool hosting.
Was using [rauchg/slackin](https://github.com/rauchg/slackin) to make shortcut for our community slack and saw that we can deploy it with now.
I started looking what the heck is `now` ?

I quickly found that `now` is a product from Zeit, so ... what is Zeit ?

So here is the Zeit motto: "Make Cloud Computing as Easy and Accessible as Mobile computing."
Looks cool ! We can host any small website and it's free with [some restrictions](https://zeit.co/pricing)

And `now` is basically a CLI interface to use Zeit, wonderfull, I love CLI !

With `now`, when you deploy a website cleanely you have 3 steps:
- `now deploy`: used to deploy local project on Zeit
- `now alias`: used to define a custom URL (I still want this website to be personal so using my name can be cool)
- `now rm {project-name}`: used to remove old version of project

# 6. Travis-CI

I found borring to make same three commands everytime, so I started looking to automate it !

Was already using Travis-CI for [weglot/weglot-php](https://github.com/weglot/weglot-php/blob/develop/.travis.yml) so the choice was fast, let's use it !

The main problem was to understand how Zeit can log us without using bundled `now` command.
And they have a solution for that: [tokens](https://zeit.co/blog/introducing-api-tokens-management) !

After that I just made a [simple travis file](https://github.com/Korbeil/website/blob/master/.travis.yml) and added `NOW_TOKEN` as encrypted variable in Travis settings.
This deployment makes:
- install ruby dependencies
- install now binary from npm
- build website with jekyll
- now deploy / alias / rm tasks

And with that, I'm caching both installed bundle dependencies and ruby installation (from [rvm](https://rvm.io/))

# 7. Outro

So with all that, I made this website !
I'll try to make smaller posts, but this was a special one !

If you would like to have more details on any part just tell me (twitter is the way !).
I'll try to make deeper presentation of the tool !