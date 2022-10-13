## Demo

Note:
This is very useful if we want to build something according to our needs. For example, if we want to
allow developers or operations to change PHP configurations or even server configurations without
giving them access to the server, we can build an interface for them, and we can utilize drupal
powerful Role-Based Access Control and its permission to secure it. And they won't have to restart
or reload the NGINX Unit.

---

## A quick recap.

Note:
We can use NGINX Unit to replace NGINX + php-fpm, with very little to lose while allowing us to gain
simplicity, flexibility, and many powerful possibilities offered by the Unit.
The configuration and behavior are similar to NGINX + FPM, so you won't face many problems
migrating.
If you are using more advanced features of NGINX that are not available yet in Unit, you can always
use NGINX as a proxy, and Unit as the upstream server.
In terms of containerization, Unit is more straightforward than NGINX + php-fpm and covers more
cases.

