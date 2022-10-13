## NGINX

- Proxy server and static file server.
- Runs in front of HTTP, FastCGI, uwsgi, or SCGI.
- NGINX + php-fpm => PHP/Drupal.

Notes:
Before we dive into NGINX Unit, let's talk about NGINX (normal NGINX). NGINX is a proxy server and
static file server and is very good at it.
Because it is a proxy server, another server needs to be run behind it either HTTP, FastCGI, uwsgi,
or SCGI. That server that runs behind is the one that runs and executes the codes, so NGINX itself
can't run and execute code or programming language.
In PHP and Drupal community, running NGINX in front of php-fpm, which acts as a FastCGI server, is
widely adopted.
In this case, FPM is the one that executes php scripts.

---

## Unit

- Application server.
- Similar to Apache with php module.
- Others: roadrunner, swoole, workerman, etc.

Notes:
Compared to normal NGINX, NGINX Unit is an application server that understands programming languages
natively including PHP. This is similar to how the php module for Apache works.
There are other PHP application servers on the rise, like roadrunner, swoole, workerman, and many
more. But most of these server changes the paradigm of PHP about not sharing state between requests.

---

## Configuration

- HTTP API.
- JSON format.
- No file.
- Graceful update.

Notes:
NGINX Unit configuration is managed dynamically over HTTP via a friendly RESTful JSON API. This
means it offers more simplicity and flexibility than NGINX + php-fpm.
By using JSON format, it is more portable because most programming languages support JSON natively.
From the DevOps standpoint, this is a huge win, because you can create custom scripts, tooling or
wrapper, and interfaces to deal with configuration easily.
Because it is an HTTP API, there are no configuration files.
And all the changes to the configuration are graceful with zero interruptions, we don't need to
restart or reload the server.

***

### NGINX Configuration

<!-- @formatter:off -->
```nginx
server {
  server_name example.com;
  root /var/www/drupal;

  location = /favicon.ico {
    log_not_found off;
    access_log off;
  }

  location = /robots.txt {
    allow all;
    log_not_found off;
    access_log off;
  }

  location ~* \.(txt|log)$ {
    allow 192.168.0.0/16;
    deny all;
  }

  location ~ \..*/.*\.php$ {
    return 403;
  }

  location ~ ^/sites/.*/private/ {
    return 403;
  }

  # Block access to scripts in site files directory
  location ~ ^/sites/[^/]+/files/.*\.php$ {
    deny all;
  }

  # Allow "Well-Known URIs" as per RFC 5785
  location ~* ^/.well-known/ {
    allow all;
  }

  # Block access to "hidden" files and directories whose names begin with a
  # period. This includes directories used by version control systems such
  # as Subversion or Git to store control files.
  location ~ (^|/)\. {
    return 403;
  }

  location / {
    try_files $uri /index.php?$query_string; # For Drupal >= 7
  }

  location @rewrite {
    rewrite ^ /index.php; # For Drupal >= 7
  }

  # Don't allow direct access to PHP files in the vendor directory.
  location ~ /vendor/.*\.php$ {
    deny all;
    return 404;
  }

  # Protect files and directories from prying eyes.
  location ~* \.(engine|inc|install|make|module|profile|po|sh|.*sql|theme|twig|tpl(\.php)?|xtmpl|yml)(~|\.sw[op]|\.bak|\.orig|\.save)?$|^(\.(?!well-known).*|Entries.*|Repository|Root|Tag|Template|composer\.(json|lock)|web\.config)$|^#.*#$|\.php(~|\.sw[op]|\.bak|\.orig|\.save)$ {
    deny all;
    return 404;
  }

  # In Drupal 8, we must also match new paths where the '.php' appears in
  # the middle, such as update.php/selection. The rule we use is strict,
  # and only allows this pattern with the update.php front controller.
  # This allows legacy path aliases in the form of
  # blog/index.php/legacy-path to continue to route to Drupal nodes. If
  # you do not have any paths like that, then you might prefer to use a
  # laxer rule, such as:
  #   location ~ \.php(/|$) {
  # The laxer rule will continue to work if Drupal uses this new URL
  # pattern with front controllers other than update.php in a future
  # release.
  location ~ '\.php$|^/update.php' {
    fastcgi_split_path_info ^(.+?\.php)(|/.*)$;
    # Ensure the php file exists. Mitigates CVE-2019-11043
    try_files $fastcgi_script_name =404;
    include fastcgi_params;
    # Block httpoxy attacks. See https://httpoxy.org/.
    fastcgi_param HTTP_PROXY "";
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_param QUERY_STRING $query_string;
    fastcgi_intercept_errors on;
    fastcgi_pass unix:/var/run/php/php-fpm.sock;
  }

  location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
    try_files $uri @rewrite;
    expires max;
    log_not_found off;
  }

  # Fighting with Styles? This little gem is amazing.
  location ~ ^/sites/.*/files/styles/ {
    try_files $uri @rewrite;
  }

  # Handle private files through Drupal. Private file's path can come
  # with a language prefix.
  location ~ ^(/[a-z\-]+)?/system/files/ {
    try_files $uri /index.php?$query_string;
  }

  # Enforce clean URLs
  # Removes index.php from urls like www.example.com/index.php/my-page --> www.example.com/my-page
  # Could be done with 301 for permanent or other redirect codes.
  if ($request_uri ~* "^(.*/)index\.php/(.*)") {
    return 307 $1$2;
  }
}
```
<!-- @formatter:on -->

Note:
Now, let's see the configurations for each of them.
This is what NGINX configuration looks like for a typical drupal site, and this is taken from their
official recipe docs.
It has its own syntax for the configuration format, so it will be hard to manipulate it
programmatically.

***

### FPM Configuration

```ini
[www]
user = _www
group = _www

listen = 127.0.0.1:9000

pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

Notes:
The next one is php-fpm configuration, which uses INI syntax. Its more simple than NGINX, and
supported by php natively, at least for reading the syntax with parse_ini_file or parse_ini_string
function, not writing it. Unfortunately, it is not very good with nested structures, unlike TOML
syntax, that very similar with INI syntax but pretty good with nested structures.

***

### Unit configuration

```json
{
  "listeners": {
    "*:80": {
      "pass": "routes"
    }
  },
  "routes": [
    {
      "match": {
        "uri": [
          "!*/.well-known/*",
          "/vendor/*",
          "/core/profiles/demo_umami/modules/demo_umami_content/default_content/*",
          "*.engine",
          "*.inc",
          "*.install",
          "*.make",
          "*.module",
          "*.po",
          "*.profile",
          "*.sh",
          "*.theme",
          "*.tpl",
          "*.twig",
          "*.xtmpl",
          "*.yml",
          "*/.*",
          "*/Entries*",
          "*/Repository",
          "*/Root",
          "*/Tag",
          "*/Template",
          "*/composer.json",
          "*/composer.lock",
          "*/web.config",
          "*sql",
          "*.bak",
          "*.orig",
          "*.save",
          "*.swo",
          "*.swp",
          "*~"
        ]
      },
      "action": {
        "return": 404
      }
    },
    {
      "match": {
        "uri": [
          "/core/authorize.php",
          "/core/core.api.php",
          "/core/globals.api.php",
          "/core/install.php",
          "/core/modules/statistics/statistics.php",
          "~^/core/modules/system/tests/https?\\.php",
          "/core/rebuild.php",
          "/update.php"
        ]
      },
      "action": {
        "pass": "applications/drupal/direct"
      }
    },
    {
      "match": {
        "uri": [
          "!/index.php*",
          "*.php"
        ]
      },
      "action": {
        "return": 404
      }
    },
    {
      "action": {
        "share": "/path/to/app/web$uri",
        "fallback": {
          "pass": "applications/drupal/index"
        }
      }
    }
  ],
  "applications": {
    "drupal": {
      "type": "php",
      "targets": {
        "direct": {
          "root": "/path/to/app/web/"
        },
        "index": {
          "root": "/path/to/app/web/",
          "script": "index.php"
        }
      }
    }
  }
}
```

Note:
Now, this is what the Unit's configuration looks like. I'm sure we all recognize this format, yeah,
it's just JSON.
And it is super easy to manipulate it programmatically.

***

### Unit

```json
{
  "listeners": {
    "*:8080": {
      "pass": "routes"
    }
  }
}
```

### NGINX

<!-- @formatter:off -->
```nginx
server {
  listen *:8080;
}
```
<!-- @formatter:on -->

Note:
Many of the directives in the Unit configuration are similar to the NGINX configuration. This
snippet, for example, is to configure the IP addresses and ports that the server will listen to.
And just like NGINX which can have multiple servers listen to different ports, Unit can also do
that.

***

### Unit

<!-- @formatter:off -->
```json
{
  "routes": [
    {
      "match": {
        "uri": ["*.engine", "*.inc", "*.install", "*.make", "*.module", "*.po", "*.profile", "*.sh", "*.theme", "*.tpl", "*.twig", "*.xtmpl", "*.yml", "*/.*", "*sql", "*.bak", "*.orig", "*.save", "*.swo", "*.swp", "*~"]
      },
      "action": {
        "return": 404
      }
    }
  ]
}
```
<!-- @formatter:on -->

### NGINX

<!-- @formatter:off -->
```nginx
location ~* \.(engine|inc|install|make|module|profile|po|sh|.*sql|theme|twig|tpl(\.php)?|xtmpl|yml)(~|\.sw[op]|\.bak|\.orig|\.save)?$ {
  return 404;
}
```
<!-- @formatter:on -->

Note:
Here we can see protection for some files, defined by patterns and returning 404.
Thing is, if you are familiar with NGINX configuration, you won't face many problems configuring
Unit.

***

### Unit

```json [5-9]
{
  "applications": {
    "drupal": {
      "type": "php",
      "processes": {
        "max": 5,
        "spare": 1,
        "idle_timeout": 10
      }
    }
  }
}
```

### FPM

```ini
[www]
pm.max_children = 5
pm.start_servers = 1
pm.min_spare_servers = 1
pm.max_spare_servers = 5
pm.process_idle_timeout = 10
```

Note:
Some parts of the Unit's configuration also have similar behavior to php-fpm.
Here, the processes are equivalent to dynamic FPM configuration. We can set the max and spare
children that will be spawned, and the process idle timeout.

***

### Unit

```json [5]
{
  "applications": {
    "drupal": {
      "type": "php",
      "processes": 20
    }
  }
}
```

### FPM

```ini
[www]
pm = static
pm.max_children = 20
```

Notes:
Or if we want to configure the Unit to behave like a static setup in FPM configuration, this is how
we do it.

***

### php Configuration

```json [5-14]
{
  "applications": {
    "drupal": {
      "type": "php",
      "options": {
        "file": "/etc/php.ini",
        "admin": {
          "memory_limit": "256M",
          "variables_order": "EGPCS"
        },
        "user": {
          "display_errors": "0"
        }
      }
    }
  }
}
```

Note:
Unit also allows us to set PHP configuration directly.
We can set the path to the PHP configuration file that we want to use.
We can also set individual configurations in the admin and user subkeys.
The admin subkey is equivalent to PHP_INI_SYSTEM mode, so can't be changed at runtime, while the
user subkey is equivalent to PHP_INI_USER mode and can be changed at runtime, eg with the ini_set
function.

***

### Environment variables

```json [5-7]
{
  "applications": {
    "drupal": {
      "type": "php",
      "environment": {
        "DATABASE_URL": "mysql://drupal@localhost:3306/drupal"
      }
    }
  }
}
```

Note:
Environment variables also can be easily configured, and they can be changed dynamically at runtime.

***

### Applying configuration

```sh
curl -X PUT --data-binary @config.json --unix-socket \
  /path/to/control.unit.sock http://localhost/config
```

```json
{
  "success": "Reconfiguration done."
}
```

Note:
To apply configuration changes, we only need to make an HTTP request to the Unit's control endpoint.
We can use widely available curl, or we can use any HTTP client out there.
One thing to note is, by default, the unit's control is using a Unix socket, so it can only be
accessed from the same machine, and not all HTTP client support Unix socket.
It can be configured to use TCP instead when starting it.
When making a request, we can upload a JSON file containing the configurations, or we can use JSON
string as a request body.

***

### Applying configuration

```json [5]
{
  "applications": {
    "drupal": {
        "admin": {
          "memory_limit": "128M"
        }
      }
    }
  }
}
```

```sh
curl -X PUT -d '"128M"' --unix-socket /path/to/control.unit.sock \
  http://localhost/config/applications/drupal/options/admin/memory_limit
```

Note:
We can either supply the whole configurations, or if you want to change only certain parts of the
configuration, you can translate the tree in the JSON into a URL path.
For example, if we want to update memory_limit in the php application options, we want to use this
path in the URL.
