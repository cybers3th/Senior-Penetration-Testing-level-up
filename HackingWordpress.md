<div align="center">

<img width="580" height="320" alt="image" src="https://github.com/user-attachments/assets/0e06bf8d-d414-4026-b0e5-fabe84e9d96f" />

# Wordpress
Wordpress by far is now one of the most chosen CMS's (Content Management System) for new companies and relatively a lot of Web Hosting companies use it.
WP (Wordpress) is written in the PHP language and most of the times runs of Apache servers with SQL dependencies in the backend.
But by having big amounts of popularity, you get huge amounts of vulnerabilities.

## Learning the Internal Structure of WP.

WP is natively allowed to be installed on Windows, Linux and MacOS systems, to exploit the server we must know the fundamentals and how the OS's work properly to tinker with a company's web server.
This research is from HTB where i learnt on it with ubuntu, which is Linux. For other systems you must practice differently.

The root of all the files from a web server in WP is at /var/www/html, which a company must have for a website to be hosted and maintained to function properly.

```
.
├── index.php
├── license.txt
├── readme.html
├── wp-activate.php
├── wp-admin
├── wp-blog-header.php
├── wp-comments-post.php
├── wp-config.php
├── wp-config-sample.php
├── wp-content
├── wp-cron.php
├── wp-includes
├── wp-links-opml.php
├── wp-load.php
├── wp-login.php
├── wp-mail.php
├── wp-settings.php
├── wp-signup.php
├── wp-trackback.php
└── xmlrpc.php

```
this what a configured WP web server looks like, clean install. (to be clarified this is learnt off HTB and not from my own teachings, if asked from htb this can be taken down.)

let's take a look at each file, file by file.

`index.php` is the homepage of WP.

`license.txt` is where information stays such as WP Versions, Hosted Web Server Version, Holder, etc etc.

`readme.html` is a reserved file for new people who do not know how WP works and how to set up, this is not so important so we can cross this out of our list

`wp-activate/php` is used for email authentication.

`wp-admin` is the file for the admin login press.
the login pages can be found at one of these files:
- `/wp-admin/login.php`
- `/wp-admin/wp-login.php`
- `/login.php`
- `/wp-login.php`

To clarify, all of these files are set as default, owner's of the web server can rename them to make it more difficult to find, or extract.

`xmlrpc.php` was a feature of WP's HTML transferring code, but it has been replaced by WP's Rest API.

the other expect for `wp-config.php` are self-explanatary.

### WP's Configuration Files and Directories

`wp-config.php` is the most important file containing data that we can use to extract, such as databases lists, usernames and passwords, auth keys etc. This file is also very important to be used to activate DEBUG mode that can be used in trouble shooting
this is one of the more important fiels to me aiming at when hacking WP structure.
```
<?php
// ** Database settings - You can get this info from your web host ** //
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'neil' );
define( 'DB_PASSWORD', 'Opera2112' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8mb4' );
define( 'DB_COLLATE', '' );

define( 'AUTH_KEY',         'some-weak-or-default-key' );
define( 'SECURE_AUTH_KEY',  'another-key' );
define( 'LOGGED_IN_KEY',    'yet-another' );
define( 'NONCE_KEY',        'nonce-key-here' );
define( 'AUTH_SALT',        'salt1' );
define( 'SECURE_AUTH_SALT', 'salt2' );
define( 'LOGGED_IN_SALT',   'salt3' );
define( 'NONCE_SALT',       'salt4' );

$table_prefix = 'wp_';

define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );

if ( ! defined( 'ABSPATH' ) ) {
    define( 'ABSPATH', __DIR__ . '/' );
}
require_once ABSPATH . 'wp-settings.php';   
```

This is an example of a `wp-config.php` file.

Common advanced configurations include debugging (define( 'WP_DEBUG', true );), post revision limits (define( 'WP_POST_REVISIONS', 3 );), and custom upload directories (define( 'UPLOADS', 'wp-content/mydir' );). Security keys should be generated using the official WordPress secret-key service to ensure uniqueness.

**Directories**

`wp-content`
The main directory for plugins, themes, and uploaded files. A prime target for enumeration, Which often contains sensitive data, vulnerable plugins, or misconfigurations leading to RCE.
```
/var/www/html/wp-content
├── index.php
├── plugins/
└── themes/
```
The uploads/ subdirectory (not shown here) is where user-uploaded files land, Here you can check for webshells, sensitive documents, or exploitable content types.

`wp-includes`
Core WordPress files, Which contains certificates, fonts, JS, widgets, and utility scripts. Everything except admin components and themes.
```
/var/www/html/wp-includes
├── theme.php
├── update.php
├── user.php
├── vars.php
├── version.php
├── widgets/
├── widgets.php
├── wlwmanifest.xml
├── wp-db.php
└── wp-diff.php
```
`version.php `— reveals the exact WP version (useful for matching CVEs).

`widgets/` — sometimes contains custom widget code with hardcoded creds or paths.

## WP's Rest API
The WordPress REST API has been part of core since version 4.7 and is enabled by default on every install. It exposes a JSON interface to essentially all of WordPress's data, such as posts, pages, users, media, comments, taxonomies, and anything a plugin registers. The base URL is /wp-json/, and hitting that directly in a browser returns the full index of available namespaces, routes, and their supported HTTP methods. 

The structure works in layers. A namespace groups related routes (e.g. wp/v2 for core, wc/v3 for WooCommerce, or something custom like my-plugin/v1). Within a namespace, each route maps to a resource type, /wp-json/wp/v2/posts is the route for posts, /wp-json/wp/v2/users for users, and so on. An endpoint is the combination of a route and an HTTP method, so GET /wp-json/wp/v2/posts (list posts) and POST /wp-json/wp/v2/posts (create a post) are two different endpoints on the same route. 

Authentication follows WordPress's standard permission model. Read endpoints for public content (published posts, pages) are accessible without login. Anything that modifies data or accesses private content requires either a valid session cookie, an application password, or a JWT token. Plugins that register their own routes sometimes get this wrong, the most dangerous misconfiguration is using __return_true as the permission callback, which means the endpoint is wide open to anyone. 

From an offensive perspective, the REST API is a high-value recon surface. The /wp-json/wp/v2/users endpoint can enumerate valid usernames without authentication, which turns a blind password spray into a targeted credential attack. The index at /wp-json/ reveals every plugin namespace and its registered routes, which tells you exactly which plugins are active and can sometimes hint at versions. Custom plugin routes that lack proper permission checks can leak sensitive data, allow unauthenticated file uploads, or even enable arbitrary code execution through chained actions. The media endpoint (/wp-json/wp/v2/media) has been a recurring source of data exposure, where files uploaded to the media library are publicly retrievable even when the surrounding pages are restricted. 

On the box itself, the REST API doesn't add much beyond what you'd already see in the filesystem, but it's worth checking the wp-includes/rest-api/ directory for any custom route registrations a developer may have added, and looking at plugin source for register_rest_route calls to identify which endpoints exist and whether their permission callbacks are sound.
</div>
