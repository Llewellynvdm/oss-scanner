# Reports for the Joomla! Security Strike Team

Use a short title that names the affected CMS component, plugin or library and describes the observed behavior.

- **Impact.** Explain who can do what, whose data or permissions are affected, and which Joomla behavior should have prevented it.
- **Tested environment.** Give the Joomla CMS version and commit. Identify the application (Site, Administrator, API or CLI) and any PHP or database details that affected the result. State which versions were tested and label any suspected version range as an inference.
- **Evidence.** Start from a fresh installation and describe any changes needed to establish the test conditions, including the user group, permissions, enabled extensions and relevant settings. Summarize the observed and expected outcomes using a small test fixture. Identify the affected source file and method.
- **Source and proposed fix.** If the finding was previously reported, cite its source. If a patch is available, explain what it corrects and give the results of any regression tests.

[JSST reporting procedures](https://developer.joomla.org/security.html) · [Joomla CMS security policy](https://github.com/joomla/joomla-cms/security/policy)
