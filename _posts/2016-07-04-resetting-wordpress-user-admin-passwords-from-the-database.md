---
layout: post
title: Resetting WordPress User/Admin Passwords from the Database
description: This is a quick and easy one, but I'm bound to forget it somewhere along the way...
---
This is a quick and easy one, but I'm bound to forget it somewhere along the way...

Resetting a WordPress user password from the database can be useful when you've misplaced the password - particularly for an admin user. Here's how you do it.

WordPress passwords are simply hashed with the `md5` algorithm, so it's pretty simple. In your console (I'm using bash here), run:

    $ echo -n newpassword | md5sum
    5e9d11a14ad1c8dd77e98ef9b53fd1ba

This is your hashed password. Make sure to include the `-n` option to omit the trailing newline from your input string.

Next, login to your mysql client:

    $ mysql -u db_username -p database_name
    Enter password: ***********

Once in, run:

    mysql > UPDATE wp_users SET user_pass = '[hash]' WHERE user_login = '[username]';

This assumes you have the `wp_` table prefix that WordPress uses by default.

For example:

    mysql > UPDATE wp_users SET user_pass = '5e9d11a14ad1c8dd77e98ef9b53fd1ba' WHERE user_login = 'admin';

You should now be able to login with your new password, in this example 'newpassword'.