# Security Policy

## Do not open a public issue

If you have found a security problem, do not file it in the tracker. A public issue tells
everyone about the hole before it is closed.

## How to report

**Use GitHub private vulnerability reporting.** On this repository, go to the **Security** tab and
choose **Report a vulnerability**. The report is visible only to the maintainer until it is fixed.

**Fallback.** If private reporting is not available to you, open a normal issue saying only that
you have a security report and asking for a private channel. Say nothing about the vulnerability
itself: the issue is public, and the point of this page is that the details never are. You will be
given somewhere private to send them.

Expect a first response within seven days.

## In scope

- The herobuilder.io site: anything that runs in the browser, including stored or reflected
  script injection through item names, class data or user input.
- The share link parser: a crafted herobuilder.io link that runs script, corrupts stored builds,
  or makes the page do something the person opening it did not ask for. Anyone can hand anyone
  else one of these links, so this is the most valuable surface on the site.
- Local storage of saved builds.
- Sign-in and accounts, once they exist. They do not exist yet. There is no login, no server-side
  build storage, and no user data held on any server we run.

## Not in scope

- **The Hero Siege game client, its servers, and its accounts.** Those belong to Panic Art
  Studios. We have no access to them and cannot act on a report about them. Take it to Panic Art
  Studios directly.
- Reports produced only by an automated scanner, with no working demonstration.
- Missing hardening headers with no attack behind them.
- Denial of service by volume of traffic.
- Social engineering of the maintainer.

## Disclosure

Report privately, give us a chance to fix it, then publish whatever you like. We will credit you
in the fix unless you ask us not to.

Hero Builder is an unofficial fan tool with no affiliation to Panic Art Studios.
