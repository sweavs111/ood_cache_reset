# Cache Reset for Open OnDemand

A small, dependency-free Passenger app that lets a user clear their own
saved Batch Connect form-value cache, for when a generated nf-core app
(or any other Batch Connect app) keeps reopening with stale defaults.

Every app [`nf2ood`](https://github.com/TuftsRT/nfcore2ood) generates links
to this app from a "Saved form values" notice in its form header
(`nfcore_ood_template/form.template.erb`):

```
/pun/sys/cache_reset/?app_slug=<slug>&return_url=<...>/session_contexts/new
```

and each generated app's `form.js` already implements the other half of the
handshake (`resetBatchConnectFormOnce`): if it's reloaded with
`?cache_reset=1` in the URL, it blanks every visible field. Until this app
exists at `/pun/sys/cache_reset/` on a given OOD instance, that link 404s -
the notice and the client-side JS were shipped ahead of the utility itself.

## Overview

Open OnDemand caches each Batch Connect app's last-submitted form values,
per user, at:

```text
~/ondemand/data/sys/dashboard/batch_connect/cache/<role>_<app_slug>.json
```

(`<role>` is `sys` or `dev` depending on where the app lives - see
`BatchConnect::App#cache_file`/`#token` in the dashboard app). This utility
lists the current user's matching cache files, deletes the one selected,
and - when reached via the `app_slug`/`return_url` link above - redirects
back with `cache_reset=1&cache_reset_at=<epoch>` appended so the
destination form's own JS clears the visible fields too.

When `app_slug` resolves to exactly one cache file, the app shows a
single-app confirm screen ("Clear the saved cache for `<slug>`?") instead
of a dropdown over every cached app - a "clear a different app's cache
instead" link (`&show_all=1`) drops back to the full list. This is still
only ever a plain GET link underneath (no side effect from loading the
page); deletion still only ever happens on `POST /clear`, same as the full
picker.

## Relationship to `TuftsRT/tufts_ood_cache_reset`

This is a port of Tufts' own reference implementation
([`TuftsRT/tufts_ood_cache_reset`](https://github.com/TuftsRT/tufts_ood_cache_reset),
MIT licensed, see [`LICENSE`](LICENSE)) - same cache directory, same
`sys_*.json`/`dev_*.json` whitelist pattern, same `app_slug`/`return_url`
query params, same `cache_reset=1&cache_reset_at=...` redirect contract.

The only real difference is the implementation: the upstream app is
Sinatra-based (`require "sinatra/base"`), which needs the `sinatra` and
`rack` gems on the Ruby Passenger runs the app under. Neither is installed
here, and `apps/sys/*` is root-owned, so there's no path to `gem install`
into wherever Passenger's app process would actually look without also
solving *that*. This port keeps the same behavior and security model but
uses only Ruby stdlib (`cgi`, `pathname`, `time`) - no gems beyond what
ships with the system Ruby, so it needs no `Gemfile`/`bundle install` step.

If your site already has `sinatra`/`rack` available to Passenger apps,
the upstream repo is a perfectly good (and more polished-looking) drop-in
alternative to this one.

## Security model

- Only lists filenames actually present in `CACHE_DIR` matching
  `\A(?:sys|dev)_[A-Za-z0-9][A-Za-z0-9_.-]*\.json\z` - nothing built from
  raw user input ever becomes a path.
- Deletion re-validates the selected filename against a freshly-discovered
  whitelist, rejects anything that resolves outside `CACHE_DIR`, and
  rejects symlinks.
- Deletion only ever happens on `POST /clear`, never on the bare `GET` this
  app is normally linked with - visiting the link just shows the picker.
- `return_url` is only ever honored if it starts with `/pun/`, contains no
  `://`, and has no `\r`/`\n` (blocks open-redirect and header-injection
  attempts); anything else falls back to `/pun/sys/dashboard`.

## Files

- `app.rb` - the app (a plain Rack-`call(env)` object, no framework)
- `config.ru` - Rack entrypoint (`run CacheResetApp.new`)
- `manifest.yml` - OOD app metadata
- `tmp/` - Passenger restarts the app when `tmp/restart.txt`'s mtime
  changes (the usual Phusion Passenger convention); not committed since
  it's a deploy-time artifact, not app source (see `.gitignore`).

## Deploying

`apps/sys/cache_reset` is typically root-owned like every other shared
`sys` app, so getting a clone in there usually needs `sudo`. If your site's
home directories are NFS-mounted with `root_squash` (root can't `chdir`
into them), clone/stage on local disk first rather than under `$HOME`,
then elevate only for the final copy:

```bash
git clone https://github.com/sweavs111/ood_cache_reset.git /tmp/ood_cache_reset
sudo mkdir -p /var/www/ood/apps/sys/cache_reset
sudo rsync -a --delete --exclude .git /tmp/ood_cache_reset/ /var/www/ood/apps/sys/cache_reset/
sudo mkdir -p /var/www/ood/apps/sys/cache_reset/tmp
sudo touch /var/www/ood/apps/sys/cache_reset/tmp/restart.txt
```

Then reload the OOD dashboard (or wait for Passenger to notice the app is
new) and confirm `/pun/sys/cache_reset/` loads.

## Testing without deploying

Because `app.rb` only depends on stdlib and reads `HOME` for `CACHE_DIR`,
it can be exercised directly with a fake `$HOME` and a synthetic Rack env,
without Passenger, Sinatra, or a real cache directory:

```ruby
ENV['HOME'] = '/path/to/fake/home'   # containing ondemand/data/sys/dashboard/batch_connect/cache/*.json
require './app'
require 'stringio'

app = CacheResetApp.new
status, headers, body = app.call(
  'REQUEST_METHOD' => 'GET',
  'PATH_INFO' => '/',
  'QUERY_STRING' => 'app_slug=nf-core-ampliseq-2-18-0&return_url=/pun/sys/dashboard',
  'rack.input' => StringIO.new('')
)
```
