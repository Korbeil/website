+++
title = "✍️ Automate PHP-CS-Fixer on PhpStorm"
date = "2018-07-20"
tags = ["php", "php-cs-fixer", "phpstorm"]
+++

Some times ago, <a href="https://github.com/curtchan">@curtchan</a> asked me if there is any articles about how to automate `php-cs-fixer` with PhpStorm.

Here we go !

Actually it's really easy, you have to:
- Have the `php-cs-fixer` phar installed inside `/usr/local/bin/` directory
- Have a `.php_cs` file on root directory of the project you wanna have this automation

If you have both, just download: <a href="/assets/phpstorm-cs-fixer-watcher.xml">this watcher</a>

And to install it on your project, go to: Preferences > Tools > File Watchers

On this page you'll find a "Import" button (green arrow going inside a square).
Click it and use the downloaded watcher.

That watcher will run `php-cs-fixer` every time you goes through a file from PhpStorm.

Simply ! :-)
