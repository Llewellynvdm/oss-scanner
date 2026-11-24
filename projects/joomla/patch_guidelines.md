# Proposed patches for Joomla CMS

Prepare a focused diff against the tested CMS revision for private JSST review. Explain the cause and why the change corrects it. Preserve the stable branch's public interfaces and existing behavior for legitimate callers. Follow the patterns of the surrounding component or library and keep unrelated formatting out of the diff.

Follow the checkout's PHP coding standards in `ruleset.xml` and `.php-cs-fixer.dist.php`. Place a focused regression test alongside the closest existing tests. Use `tests/Unit` for isolated PHP behavior, `tests/Integration` for interactions between services, or the relevant `tests/System` suite when the behavior is visible only through a running site.

Include testing instructions, the result before the patch, the expected result after it, and the checks actually run. Where permissions are involved, cover both the restricted case and a user who should retain access. Identify AI-assisted changes as proposals so a Joomla contributor can review and validate them under the project's policy.

[CMS contribution guide](https://github.com/joomla/joomla-cms/blob/6.2-dev/.github/CONTRIBUTING.md) · [CMS pull request template](https://github.com/joomla/joomla-cms/blob/6.2-dev/.github/PULL_REQUEST_TEMPLATE.md) · [Generative AI policy](https://developer.joomla.org/generative-ai-policy.html)
