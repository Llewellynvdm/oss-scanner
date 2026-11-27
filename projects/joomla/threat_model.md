# Threat model: Joomla! CMS

The scanner reads this file before it starts. It covers Joomla! CMS as developed on the default branch of
https://github.com/joomla/joomla-cms, which is always the patch branch of the current stable minor (6.2-dev when this
was written): the components, modules, plugins, templates, libraries and web services that a Joomla release ships,
the installation application and the console. The Joomla! Security Strike Team (JSST) assesses and classifies what
the scanner finds and coordinates the fix; its procedures and the levels it uses are at
https://developer.joomla.org/security.html.

## What this project does and where untrusted input enters

Joomla is a PHP content management system on MySQL/MariaDB or PostgreSQL. One checkout serves four applications that
share the code under `libraries/` and one database:

| Application | Where | Who uses it |
| --- | --- | --- |
| Site | `index.php`, `components/`, `modules/`, `plugins/`, `templates/cassiopeia/`, `layouts/` | everyone: anonymous visitors and logged-in users |
| Administrator | `administrator/index.php`, `administrator/components/`, `administrator/modules/`, `administrator/templates/atum/` | Manager, Administrator and Super Users |
| Web services API | `api/index.php/v1/...`, `api/components/`, `plugins/webservices/`, `plugins/api-authentication/` | holders of an API token, with their group's permissions |
| Console | `cli/joomla.php`, `libraries/src/Console/` | the server operator |

The installation application (`installation/`) runs only before a site is installed; a release deletes it afterwards.
This development checkout keeps it, as Joomla's development branches do, and serves the installed site regardless.

The CMS library is `libraries/src/` (namespace `Joomla\CMS`). The Joomla Framework packages under
`libraries/vendor/joomla/` (database, filter, input, session, router, http and others) are the project's own code and
in scope; the JSST handles their fixes too. Everything else under `libraries/vendor/` and `media/vendor/` is
third-party. JavaScript and CSS are built from `media_source/` into `media/`: a finding in a built file belongs to
its source.

Untrusted input enters through:

- **Every HTTP request** to the site, the administrator or the API: the path and SEF route, query and POST data
  (`option`, `view`, `task`, `layout`, `format`, `tmpl`, `Itemid`, `return`, list filters and `list[ordering]`),
  cookies, the `Host` and proxy headers, and uploaded files.
- **Content stored by one user and shown to another**: articles, categories, tags, contacts, banners, newsfeeds, menu
  items, custom field values (`com_fields`, `plugins/fields/`), user names and profile data, user notes, redirects,
  mail templates, and the site, template and extension settings an administrator edits. Super Users and administrators
  see what lower-privileged users wrote, in lists, in editors and in previews.
- **Text filtering**: what a user may store depends on their group's text filter (Global Configuration › Text Filters,
  applied through `Joomla\CMS\Filter\InputFilter`). The stock filters are in the table below. Getting script past a
  group's filter is a finding; so is output that escapes nothing where the filter did not run.
- **Files**: media manager uploads (`com_media`, `plugins/filesystem/local`, `MediaHelper`) and the extension,
  language pack and template file uploads that Super Users make.
- **Authentication flows**: login (site, administrator and API), the remember-me cookie, registration and account
  activation, password reset, multi-factor authentication (`plugins/multifactorauth/`, `plugins/system/webauthn`) and
  API tokens (`plugins/user/token`, `plugins/api-authentication/token`).
- **Server-side requests**: update servers and the Install from URL feature (Super Users), the Joomla update itself,
  which TUF protects (`php-tuf`), and the HTTP client in `libraries/vendor/joomla/http`.

### Trust model

Joomla's access control is user groups, access levels and per-asset permissions
(`administrator/components/*/access.xml`, the `#__assets` table, `Joomla\CMS\Access`). A stock installation seeds
these groups and text filters:

| Group | Trust | Text filter |
| --- | --- | --- |
| Public, Guest (anonymous) | Untrusted. | No HTML |
| Registered | Can log in to the site and edit their own profile. Untrusted beyond that. | No HTML |
| Author, Editor, Publisher | The site content workflow: create own content; edit all content; publish. No administrator login. | Default Forbidden List |
| Manager | Administrator login; manages articles, categories, featured content and media (`core.manage` on `com_content` and `com_media`); not menus, users, extensions or Global Configuration. | Default Forbidden List |
| Administrator | `core.manage` on every component and `core.admin` on most: manages users (not Super Users), menus, plugins, templates and component options. Cannot install extensions (`com_installer` denies it), change Global Configuration or touch Super Users. Escalation from Administrator to Super User or to code execution is a finding. | Default Forbidden List |
| Super Users | Trusted with the whole site. Install extensions, edit template files, change any setting: each runs code by design. A finding that needs Super User access is not a vulnerability. | No Filtering |

The server operator is trusted: `configuration.php`, the file system, the database and the console. Third-party
extensions are out of scope and go to the Vulnerable Extensions List (https://vel.joomla.org/). Third-party libraries
go upstream; report them here only when Joomla's use of a library creates the vulnerability.

## Components that matter most / least

Highest priority:

1. **Access control.** `authorise()` and `canDo` checks in controllers and models, access levels on items and
   categories, `core.edit.own` against other users' items, the API's parity with the administrator (every endpoint
   under `api/components/` must check what its administrator counterpart checks), user and group management
   (`com_users`: who may edit whom, assign which groups, reset which passwords), batch and checkin tasks, and the
   `#__assets` tree itself.
2. **Cross-site scripting.** Output that bypasses `htmlspecialchars` and `HTMLHelper` escaping, `Text::_()` strings
   built from input, the text filter configuration and `InputFilter`, SVG uploads (`enshrined/svg-sanitize` as Joomla
   uses it), custom field rendering, the media manager and its previews, the TinyMCE and CodeMirror integrations,
   JavaScript in `media_source/`, mail templates, error pages and template overrides.
3. **SQL injection.** The query builder in `libraries/vendor/joomla/database` as the CMS uses it: concatenated input
   in `where()` and `order()`, list ordering and direction outside `filter_fields`, `escape()` misuse, Smart Search
   (`com_finder`) query building, and the PostgreSQL driver paths, which this image cannot run.
4. **Files.** Path traversal and extension or MIME checks in the media manager (`upload_extensions`,
   `restrict_uploads`, `check_mime`, `image_extensions`), `Joomla\CMS\Filesystem` and `libraries/vendor/joomla/filesystem`,
   archive extraction in the installer (`libraries/vendor/joomla/archive`), the template file editor, and `tmp/` handling.
5. **Authentication and sessions.** The login flows of all three web applications, the remember-me cookie, password
   reset and activation tokens, multi-factor enrolment and verification, WebAuthn, API token checks, session storage
   (`#__session`, `libraries/vendor/joomla/session`), CSRF tokens (`Session::checkToken()`, `HTMLHelper::_('form.token')`)
   on every state change, and open redirects through `return` parameters (`Uri::isInternal()`).
6. **Deserialisation and mass assignment.** `unserialize()` on anything an attacker influences, `Registry` and JSON
   params parsing, `Table::bind()` and form filtering that could set fields a user may not set (`block`, `groups`,
   `sendEmail`, `activation`), and `filter="raw"` form fields.
7. **Server-side requests.** Fetches of update servers, Install from URL and other URLs an administrator configures;
   the SSRF question is whether a lower-privileged user can make the site fetch a URL of their choice.

Lower priority or out of scope:

- `build/`, `tests/`, `node_modules/`, `media_source/` and `.oss-scanner/`: not in a release. Apache
  denies them here. `installation/` is in scope only as the installer of an uninstalled site, which this image is not.
- `libraries/vendor/` other than `joomla/*`, and `media/vendor/`: upstream, unless Joomla's use creates the issue.
- The sample data plugins (`plugins/sampledata/`) and the Cypress System tests under `tests/System/`.
- Third-party extensions and live Joomla sites.

## How to exercise it

- Run `joomla-start` first. It starts MariaDB and Apache (the container has no init system) and is safe to run again.
  `joomla-stop` stops both. The site is `http://127.0.0.1/`, the administrator `http://127.0.0.1/administrator/`, the
  API `http://127.0.0.1/api/index.php/v1/`. Apache listens on the loopback interface only.
- The site is installed with the stock profile and no sample data. One account per core group exists; every password
  is `<login>-password`:
  - `superuser` (Super Users), `administrator` (Administrator), `manager` (Manager), `publisher` (Publisher),
    `editor` (Editor), `author` (Author), `registered` (Registered); anonymous is the Guest group.
  - Two settings differ from a stock site so that each account can use the API with its group's permissions:
    Registered and Manager (and the groups under them) have `core.login.api` and `core.api`, which a stock site grants
    Super Users alone, and every group may hold an API token. Name this prerequisite when a finding depends on it.
- `joomla-api-token <login>` prints a user's API token, creating one if they have none. For example:
  `curl -H "Authorization: Bearer $(joomla-api-token editor)" http://127.0.0.1/api/index.php/v1/content/articles`.
  Basic authentication for the API is disabled, as on a stock site.
- Emails are not sent; each one lands in `/var/mail/joomla/` as an `.eml` file, so password reset links,
  activation links and multi-factor codes can be read.
- The database is `joomla` on MariaDB, user `joomla` / `joomla`, table prefix `jos_`, over the unix socket;
  `mariadb joomla` opens a client as root. `joomla_test` is for the Integration suite. The configuration is
  `/src/configuration.php`.
- `joomla-console <command>` runs `cli/joomla.php` as www-data: `user:add`, `user:addtogroup`, `user:list`,
  `extension:list`, `config:get`, `config:set`, `cache:clean`, `site:down`, `site:up`, `tasks:run` and the others
  `joomla-console list` shows. `as-www-data <command>` runs anything else as the web server's user.
- Patches go to `/src`. A PHP change takes effect on the next request. A JavaScript or SCSS change goes to
  `media_source/` and is rebuilt offline with `as-www-data npm run build -- -a` (everything) or
  `as-www-data npm run build -- -n <resource>` (one resource).
- Tests: `joomla-phpunit --testsuite Unit` runs the CMS unit suite, `joomla-phpunit --testsuite Integration` the
  integration suite against `joomla_test` (its LDAP tests are skipped: there is no LDAP server). Both run offline. The
  Cypress System suite needs a browser and does not run here; reproduce through curl or PHP instead. The build runs
  both PHPUnit suites and reports, but does not fail on, test failures.
- Logs: PHP and Apache errors in `/var/log/apache2/error.log`, requests in `/var/log/apache2/access.log`, Joomla's
  own logs in `/src/administrator/logs/`. Debug mode and error display are off, as on a production site.
- Stock settings worth knowing: user registration is off (Users › Options); `htaccess.txt` is not active (copy it to
  `/src/.htaccess` to test the recommended production configuration); the HTTP Headers plugin is enabled with its
  defaults; Smart Search indexing has not run.

## How you rate severity

The JSST classifies a confirmed issue by impact (who can do what to a site) and severity, and publishes the levels it
uses at https://developer.joomla.org/security.html; every Joomla security announcement states its Impact, Severity and
Probability. For the scanner's severity field use the scale below, worded in Joomla's terms. The JSST assigns the final
classification and may move an issue up or down.

- **Critical**: anonymous visitors or Registered users can run code on the server, inject SQL, bypass authentication
  or take over a Super User account, in a stock configuration.
- **High**: SQL injection, authentication bypass or account takeover that needs a site content group (Author, Editor,
  Publisher) or a common non-default setting; stored XSS that anonymous, Registered or site content users can plant
  where Managers, Administrators or Super Users see it; escalation from Manager or Administrator to Super User or to
  code execution; reading or changing other users' private data at scale.
- **Medium**: XSS that needs Manager or Administrator rights or an uncommon setting; CSRF on a state-changing action;
  access control slips that expose unpublished or access-restricted content or let a content group act outside its
  permissions; path traversal that reads non-sensitive files; open redirects; denial of service from a single request.
- **Low**: findings that need a configuration most sites do not run, information disclosure of little value, and
  self-XSS.

Always state the attacker's starting position in the group table's terms, whether the stock configuration suffices,
and any setting, extension or permission the finding depends on. Assess the stock installation first.

## Anything to leave alone

- Anything that needs Super User access, and what Super Users can do by design: unfiltered HTML, the template file
  editor, extension installation and Install from URL.
- Version disclosure: `administrator/manifests/files/joomla.xml`, the generator meta tag, `README.txt`, `LICENSE.txt`,
  `htaccess.txt`, `web.config.txt` and `robots.txt.dist` are public by design.
- Missing or weak HTTP security headers, the absence of login rate limiting or account lockout, and other hardening
  that the site owner configures (the HTTP Headers plugin, `.htaccess`, the web server).
- Findings that need `$debug = true`, `display_errors`, maximum error reporting, No Filtering for a group below Super
  Users, or the upload restrictions (`restrict_uploads`, `check_mime`) turned off: settings Joomla documents as unsafe.
- The directories a release does not ship (`build/`, `tests/`, `node_modules/`, `.oss-scanner/`) and, on this
  development checkout, the presence of `installation/`.
- Third-party extensions (the Vulnerable Extensions List), third-party libraries under `libraries/vendor/` and
  `media/vendor/` unless Joomla's use of them is the problem, and volumetric denial of service.

The two sections above describe the JSST's published practice as this enrolment's authors understand it; the JSST
reviews and corrects them.
