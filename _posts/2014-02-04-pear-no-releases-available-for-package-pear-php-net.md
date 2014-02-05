---
layout: post
---
This one is more of a note to self than anything else, but may help others as it's not that clear. 

I was recently trying to install [PHP CodeSniffer](http://pear.php.net/package/PHP_CodeSniffer) on a machine using [PEAR](http://pear.php.net/), but kept receiving the following error message:


    > pear install PHP_CodeSniffer
    No releases available for package "pear.php.net/PHP_CodeSniffer"
    install failed

Turns out that clearing PEAR's cache seems to do the trick:

    > pear clear-cache
    reading directory ...
    242 cache entries cleared

Running the install command then worked as expected:

    > pear install PHP_CodeSniffer
    downloading PHP_CodeSniffer-1.5.1.tgz ...
    ...
    ...
