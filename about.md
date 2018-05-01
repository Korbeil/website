---
layout: page
title: About
permalink: /about/
---

# In short

I'm Baptiste Leduc, a Web Engineer at [Weglot](https://weglot.com/), Paris, France.
I started my career with Software Development at [Alsim Simulateurs](https://www.alsim.com/) and I fall quickly into the Web.
Found my way by making websites with various technologies (such as Laravel or Symfony).

Apart from all that, I like to cook & wakeboarding a lot !
And when I still have time I'm enjoying improving my music skills (mostly piano and MPC).

---

# Experiences

## Resume

- [2018 -> Now // Weglot - Integrations Developer](#weglot---integrations-developer)
- [2016 -> 2018 // mon-avocat - CTO](#mon-avocat---cto)
- [2014 -> 2016 // Sellsy - Backend Developer](#sellsy---backend-developer)
- [2011 -> 2013 // Alsim Simulateurs - Developer](#alsim-simulateurs---developer)

---

## [Weglot](https://weglot.com/) - Integrations Developer
Weglot is a SaaS translation platform. The goal is to plug onto everything easily as possible & to deliver high quality automated translations through machine learning and usual Bing/Google/Yandex APIs.

---

## [mon-avocat](https://www.mon-avocat.fr/) - CTO
mon-avocat.fr is an online search engine to find lawyers close by either according to the domain involved by visitor's search.

### Framework migration
When I started to work at mon-avocat.fr, I was the only developper and the website was managed by a freelance developper with Zend Framework 1. Due to LTS ending 28/09/2016 and that I wasn't familiar with this framework. I choosed to switch to Laravel and to make a stronger code base. In 3 month we switched the search engine (which was the online services we got at this time), made a new administration UI and made a new graphic UI for our frontend. Since that we using Laravel 5.*

### Scoring managment
Scoring is, basically, in which order we should list lawyers when some visitors hit the search button. We made a lot of work on that subject. At start there was no custom ordering, only alphabetic order was used and was clearly in favor of lawyer that have a name starting wiith first alphabet letters.
Today, we're handling scoring with a much smother way. We divided scoring in two parts: static which estimate the number of leads a lawyer got and dynamic which mostly depends on visitor position when he hits the search button.

### Search engine overhaul
With more and more hits, we had to focus onto our core service: the search engine. At first he was based on a simple but efficient SQL query. For sure, with traffic increase, this was'nt fast enought and started to be slow due to multiple concurrent hits. So we started looking into the famous ElasticSearch to index all our lawyers, their position, score and all other needed informations in lawyers listing. With ES implementation, we reduced search engine time by 3.

### GraphQL API
After the first graphic overhaul we stated that we needed to get a new branding and so a real analysis of our frontend. So we started rethink our frontend by focusing on emergent technologies like VueJS. Since VueJS is Javascript-based, we used facebook's GraphQL logic to make an API to connect backend & frontend. GraphQL permit us to make simple queries to get everything frontend needs in a short time. But we choosed to implement it through PHP and not usual node.js implementation. So we got some of the features that made GraphQL (such as subscription), not available."

---

## [Sellsy](https://welcome.sellsy.com/) - Backend Developer
Sellsy is a company of about 30 employees. They sell an innovative cloud CRM & ERP suite for small and medium business.

### Overhaul of "Subscription" module
This module allows to create invoices at defined intervals (annual / monthly / half-yearly / ...) as a model. Before we based on existing invoices. I set up a model system to have a solid basis for subscriptions and have allowed the use of "tags" in these models to give the dates of upcoming subscription or subscription beach dynamically.

### Handling incoming emails "Parsemail"
Overhaul of the management of incoming emails. Originally it consisted of a parsemail procedural file I refactored in a class (the object paradigm shift to better separate the various components of the tool) and then adding a deep management emails encodings (which was not managed before).

### Production Architecture Migration:
Getting Started folder of the migration of our production servers to a new architecture (including a load-balancer). Intermediary between outsourcing and Sellsy. Placing applications and consolidation of the configuration management into a centralized configuration.

### Git & CI migration
I started to setup gitlab server and to define a correct workflow for us.
After that we migrate small repositories to the new gitlab infrastructure and we started to document all repositories (atleast with a README.md file).
Then I wrote a script to use CI on one of our repository (as a pilot to, in future, apply that workflow to our main repository)

---

## [Alsim Simulateurs](https://www.alsim.com/) - Developer
Alsim Simulators is a company of about 25 employees, they sell flight simulators designed for flight schools worldwide. The best known of their product is ALX

### Establishment of a stock management application
Part-time between stocks and development, I analyzed the need and existing on inventory management and implemented a management application to change the existing.

### Design and development of a driving simulator application on an iOS environment
Based on the existing engine simulator, I created an iPad application allowing simple flight simulation (single card and single aircraft model). To this I added a voice server management to simulate the control towers (with Mumble)