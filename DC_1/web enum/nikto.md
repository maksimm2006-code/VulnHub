nikto -h http://192.168.96.25

	+ Server: Apache/2.2.22 (Debian)
	+ ERROR: Failed to check for updates: 403
	+ [999986] /: Retrieved x-powered-by header: PHP/5.4.45-0+deb7u14.
	+ No CGI Directories found (use '-C all' to force check all possible dirs). CGI tests skipped.
	+ [999984] /robots.txt: Server may leak inodes via ETags, header found with file /robots.txt, inode: 152289, size: 1561, mtime: Thu Nov 21 00:45:59 2013. See: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
	+ [999996] /robots.txt: contains 36 entries which should be manually viewed. See: https://developer.mozilla.org/en-US/docs/Glossary/Robots.txt
	+ [500170] /misc/favicon.ico: identifies this app/server as: Drupal 7.x. See: https://en.wikipedia.org/wiki/Favicon
	+ [600050] Apache/2.2.22 appears to be outdated (current is at least 2.4.66).
	+ [600625] PHP/5.4.45-0+deb7u14 appears to be outdated (current is at least 8.5.1).
	+ [013587] /: Suggested security header missing: content-security-policy. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
	+ [013587] /: Suggested security header missing: x-content-type-options. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options
	+ [013587] /: Suggested security header missing: strict-transport-security. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security
	+ [013587] /: Suggested security header missing: permissions-policy. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Permissions-Policy
	+ [013587] /: Suggested security header missing: referrer-policy. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy
	+ [999967] /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
	+ [999972] /: DEBUG HTTP verb may show server debugging information. See: https://docs.microsoft.com/en-us/visualstudio/debugger/how-to-enable-debugging-for-aspnet-applications?view=vs-2017
	+ [001384] /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP Easter Eggs reveals potentially sensitive information via HTTP requests that contain specific QUERY strings. See: https://labs.detectify.com/writeups/do-you-dare-to-show-your-php-easter-egg/
	+ [001385] /?=PHPE9568F36-D428-11d2-A769-00AA001ACF42: PHP Easter Egg reveals potentially sensitive information via HTTP requests that contain specific QUERY strings. See: https://labs.detectify.com/writeups/do-you-dare-to-show-your-php-easter-egg/
	+ [001386] /?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP Easter Egg reveals potentially sensitive information via HTTP requests that contain specific QUERY strings. See: https://labs.detectify.com/writeups/do-you-dare-to-show-your-php-easter-egg/
	+ [001387] /?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP Easter Egg reveals potentially sensitive information via HTTP requests that contain specific QUERY strings. See: https://labs.detectify.com/writeups/do-you-dare-to-show-your-php-easter-egg/
	+ [001906] /user/: This might be interesting.
	+ [999100] /README: Uncommon header(s) 'tcn' found, with contents: choice.
	+ [003569] /INSTALL.txt: Default file found.
	+ [003570] /UPGRADE.txt: Default file found.
	+ [003571] /install.php: Drupal install.php file found. See: https://drupal.stackexchange.com/questions/269076/how-do-i-restrict-access-to-the-install-php-filehttps://drupal.stackexchange.com/questions/269076/how-do-i-restrict-access-to-the-install-php-file
	+ [003572] /install.php: install.php file found.
	+ [003573] /LICENSE.txt: License file found may identify site software.
	+ [003575] /xmlrpc.php: xmlrpc.php was found.
	+ [003577] /INSTALL.mysql.txt: Drupal installation file found. See: https://drupal.stackexchange.com/questions/269076/how-do-i-restrict-access-to-the-install-php-file
	+ [003578] /INSTALL.pgsql.txt: Drupal installation file found. See: https://drupal.stackexchange.com/questions/269076/how-do-i-restrict-access-to-the-install-php-file
	+ [003584] /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/


