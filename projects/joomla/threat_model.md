# Joomla! CMS

Joomla publishes content through the site application and provides an
administrator application, a web services API and a console. The scope of this
enrollment is the selected revision of `joomla/joomla-cms`, including the
components, modules, plugins and templates distributed with the CMS.

## Permissions and data

Sites define their own user groups, access levels and permissions. An account
can have permission to sign in without having permission to read, create, edit
or publish a particular item. Core should enforce the site's configured access
rules on every supported interface, including requests made directly to the
API. A user entrusted with a specific administrative task should retain only
the permissions granted for that task.

Assess the behavior of a stock installation first. Where a finding depends on
an optional core feature, configuration setting or custom group permission,
identify that prerequisite. Distinguish defects in Joomla's use of a dependency
from defects that occur entirely in that dependency. Independently distributed
extensions and live Joomla installations are outside this enrollment.

## Local checks

The source checkout and the built media assets are in `/src`. Run
`./libraries/vendor/bin/phpunit --testsuite Unit` there for the CMS unit suite.
`joomla-smoke` installs a disposable copy with MariaDB, checks the CMS console
and verifies that Apache serves the checkout's `LICENSE.txt`, then removes the
installation. Both checks run offline.

## Assessment and reporting

The Joomla Security Strike Team assesses impact and severity separately.
Describe the affected data or operation, the permissions required, and the
effect on the site owner or user. The team assigns the final classification.
The JSST coordinates private review and publishes confirmed issues after a
release containing the fix.

- [Joomla Security Strike Team threat levels](https://developer.joomla.org/security.html)
- [Joomla CMS private reporting policy](https://github.com/joomla/joomla-cms/security/policy)
