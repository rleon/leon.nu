+++
date = "2014-06-14"
draft = false
title = "dot files in nginx"
slug = "deny-access-to-files-starting-with-a-dot-location-in-nginx"
description = "nginx configuration tip"
tags = ["nginx", "tip"]
+++

Deny access to files starting with a dot is a common task. It can be done by editing main configuration file (nginx doesn't allow configuration on the fly).

Such configuration file is usualy located under `/etc/nginx/conf.d/`

Add the following lines there.
```nginx
# Deny access to files starting with a dot
location ~ /\. {
	deny all;
}
```
save and restart nginx `sudo service nginx restart`.

The deny directive is working as regular expression and it will prohibit access in subdirectories as well.
