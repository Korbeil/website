---
layout: post
title:  "My local server with the Symfony binary"
date:   2019-03-21 10:42:00 +0200
categories: symfony php server docker
---

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/my-local-server-with-the-symfony-binary) which is also available [in french](https://jolicode.com/blog/mon-serveur-local-avec-le-binaire-symfony).

In order to develop efficiently and to be able to foresee production issues as quickly as possible, it is a good thing to have a local stack as close as possible to the production one. For this reason, Docker is the tool that we strongly recommend (and so for many years now!).

However, we also need the development environment to be the fastest possible. As we all experienced, Docker and MacOS are clearly not BFF’s in that regard. For a while, we have been running our yarn scripts out of Docker containers specifically for our MacOS’s users.

We started today to use locally the Symfony binary, and it has been a real rebirth for these users. Some of them now experience execution times approaching 1 second, versus 15 seconds with Docker.

# The Symfony binary? 🤔

It is an evolution of the Symfony phar that was presented at the Lisbon SymfonyCon in December 2018. Rewritten entirely in Go, it allows us to create our Symfony projects but also to have a local development server (or a remote one through Symfony Cloud). Thanks to the power of Go, the development server now includes many new tools that we will discover through this article!

# How does it work ? ⚙

First of all, it is necessary to install [the Symfony binary](https://symfony.com/download). We will also need to have PHP installed locally to run the server.

Then, just go to the root of the project and run:

<pre><code class="language-sh">symfony serve --no-tls</code></pre>

The `--no-tls` option is used to launch the server without HTTPS, we will see this configuration later. The binary will give you the ip and port on which the application runs and that's it! 🍾

Thanks to this, we now have a local website that runs with good performances - for development purposes only, of course.

But we usually have a more complete stack in development environment: several applications, domain names, https,...

# Let's add some nice domain names! 💅

The binary allows us to manage a proxy system to give domain names to our applications. To do this, you must first attach a domain name to our application:

<pre><code class="language-sh">symfony local:proxy:domain:attach foo.bar</code></pre>

⚠ All domain names will be suffixed by `.wip` by default. In our case, we will use `foo.bar.wip` as our domain.

Then just launch the proxy:

<pre><code class="language-sh">symfony local:proxy:start</code></pre>

The command will display the proxy address. If you access it in a browser, you will find the list of domain names and the directories to which they point.

![Symfony CLI Proxy Overview](https://jolicode.com/media/original/2019/IBXIG44.png)

We can see here 2 applications with their domain names. The servers are currently stopped, but, as seen above, a simple `symfony serve` command will be enough to launch them.

In order for our browser to access these domain names, we must add the proxy address by adding `/proxy.pac` to our system preferences. Detailed examples of this configuration are available in the [presentation slides](https://speakerdeck.com/fabpot/symfony-local-web-server-dot-dot-dot-reloaded?slide=32) of the tool.

And that's it, our site is available with the address we chose! 🎉

# HTTP is ugly, can't we have HTTPS? 🔐

The issue with creating a certificate manually is that browsers will block self-signed certificates. We will use the Symfony binary to create a local certification authority. To do this, use the following command only once:

<pre><code class="language-sh">symfony local:server:ca:install</code></pre>

And that's it, we just have to run the server (without the `--no-tls` option this time) and we will have our application available in HTTPS. 👌

<pre><code class="language-sh">symfony local:server:start</code></pre>

# And how do I see what's going on on my site? 🕵

Having your website online is good, but generally, we won't leave the command `local:server:start` in the foreground and we will prefer to put it in the background with the `--daemon` option. As a result, we will loose logs… 😓

The Symfony binary includes a command for logs that will combine PHP, Symfony and the HTTP server in the same place:

<pre><code class="language-sh">symfony local:server:log</code></pre>

By default, we will have everything in the command but it is possible to filter using the `--no-app-logs` or `--no-server-logs` options.

# Conclusion

We looked at this binary following performance concerns (mainly under MacOS) on one of our projects that has several applications running under Docker.
Thanks to a hybrid solution based on Docker for the data and services parts (redis, mysql/postgresql and rabbitmq) and Symfony binary for the servers of the different applications, we have managed to guarantee for all developers (Linux or MacOS) a good execution speed, even for a highly complex project.
If you are interested in knowing more, feel free to send us a little tweet, and we will write a more complete article on this hybrid solution.
