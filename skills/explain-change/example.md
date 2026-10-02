# Example report

A complete report for a small change with two purposes. The request was given in chat: "Users say password reset links often show 'Link expired' right after they get the email. Fix it. Also let admins set the link lifetime in the site settings; keep 1 hour as the default." Coding notes in `.change-notes/reset-links.md` record the cause, a constraint, a rejected option and three runs.

````markdown
# Fix reset links that expire at once, and make their lifetime a setting

Range `5b1e0c2..9d4f7a3`.

## Summary

**Request and result.** Reset links no longer expire at once, and admins can set their lifetime in the site settings, with 1 hour as the default. One point is a reading of the request: the setting takes whole hours, so admins cannot set 30 minutes. Decide whether whole hours are enough. See [Link lifetime setting](#link-lifetime-setting).

### What changed

[Reset links that expire at once](#reset-links-that-expire-at-once)

- A reset link previously expired at once when the server clock was behind UTC, as on the production server in `America/New_York`. It now stays valid for its lifetime.

[Link lifetime setting](#link-lifetime-setting)

- The reset link lifetime was previously fixed at 1 hour. Admins can now set it in whole hours on the site settings page.
- A link that was already sent previously kept its 1 hour. It now follows the current setting: if an admin makes the lifetime shorter, older links stop working at once.

The change makes `accounts` depend on `admin`, which already depends on `accounts`. See [Link lifetime setting](#link-lifetime-setting).

**Review first.**

- The fix leaves the cause. `make_token` still stores local time, and invite links still compare it with UTC, so they expire 4 or 5 hours early. The agent kept `make_token` unchanged so that invite behavior stays the same. In my judgment, this constraint costs more than it gives. See [Reset links that expire at once](#reset-links-that-expire-at-once).
- The new test passes without the fix, because all tests run with `TZ=UTC`. So no test guards against the return of the problem. See [Reset links that expire at once](#reset-links-that-expire-at-once).
- The request did not ask for the new dependency of `accounts` on `admin`. In my judgment, the setting belongs in `core.settings`, which the project already uses for module settings. See [Link lifetime setting](#link-lifetime-setting).
- The lifetime applies when a link is checked, not when it is sent. The agent considers it better to store the expiry time in each token, but did not do it, to avoid a migration of `Token`. See [Link lifetime setting](#link-lifetime-setting).

**What ran.** Per notes, the agent reproduced the expired link with `TZ=America/New_York` before the fix and saw a valid link after it. After the last edit, the agent ran `pytest`: 48 tests passed. These tests ran on SQLite, because Docker was not available and the PostgreSQL container did not start. So the new migration `admin/migrations/0007_sitesettings_reset_link_hours.py` and the tests did not run on PostgreSQL. In my judgment, this blocker is real: the recorded error says that the Docker daemon is not available, and the change does not touch Docker. To complete the check, run `docker compose up -d db`, then `python manage.py migrate` and `pytest`.

## Reset links that expire at once

`make_token` stores `created_at` in the local time of the server, but `is_expired` compared it with `datetime.utcnow()`. The production server runs in `America/New_York`, 4 or 5 hours behind UTC. So a new reset link looked 4 or 5 hours old and expired at once. The cause is in `make_token`: all tokens get their time there, and all checks expect UTC.

The fix changes only `is_expired`: it now compares with `datetime.now()`. This removes the symptom for reset links, but the cause stays. Invite links also get their time from `make_token`, and `invites/views.py:41` still compares it with `datetime.utcnow()`. So an invite link, which must live 48 hours, expires 4 or 5 hours early.

Per notes, the agent kept `make_token` unchanged so that invite behavior stays the same. But the present invite behavior is the same error. If `make_token` stored `datetime.utcnow()`, both checks would be correct, and `is_expired` would need no change. Reset links sent in the hour before the deploy would then expire at once, and old invite links would lose the same 4 or 5 hours as now. In my judgment, this constraint costs more than it gives: it is the reason why the cause stays, and its only benefit is that an error in invites stays the same.

**Check by hand.** Start the app with `TZ=America/New_York`, ask for a password reset for a test user, and open the link from the email at once. Before the fix, the page says "Link expired". After the fix, it shows the new password form. Per notes, the agent ran these steps before and after the fix.

`tests/test_reset.py:31` adds `test_reset_link_valid_right_after_creation`. It creates a token and checks that `is_expired` returns `False`. `pytest.ini` sets `TZ=UTC` for all tests, so `datetime.now()` and `datetime.utcnow()` give the same time, and the test passes without the fix too. The notes do not record a run without the fix. To guard the fix, the test must set a time zone behind UTC itself, because `pytest.ini` replaces `TZ`.

### Implementation details

```diff
 def is_expired(token):
-    age = datetime.utcnow() - token.created_at
+    age = datetime.now() - token.created_at
     ...
```

`make_token` sets `created_at=datetime.now()` (`accounts/tokens.py:12`). `invites/views.py:41` checks `datetime.utcnow() - token.created_at > INVITE_LIFETIME`. `pytest.ini:4` sets `env = TZ=UTC`.

## Link lifetime setting

`SiteSettings` gets the field `reset_link_hours`, a whole number of hours with the default 1, and the site settings page shows it. The agent read "link lifetime" as whole hours. For shorter links, such as 30 minutes, the field must hold minutes. The field also accepts 0, which makes each reset link expire at once.

`is_expired` reads the setting at each check, so a link follows the current setting, not the setting at the time when it was sent. If an admin changes the lifetime from 24 hours to 1 hour, links sent 2 hours ago stop working at once. A longer lifetime makes old links live longer. Per notes, the agent considers it better to store an `expires_at` time in each token, so that a link keeps the lifetime that it was sent with. The agent did not do it, because it needs a migration of the `Token` table. To switch now costs the same migration, and nothing else relies on the present behavior. In my judgment, the loss is small, because admins rarely change the lifetime, so the switch can wait until someone needs it.

`accounts` now imports `admin.models.SiteSettings`. Before the change, only `admin` imported `accounts`: `admin/users.py:3` imports `accounts.models.User`. So the two modules now depend on each other. The agent imports `SiteSettings` inside `reset_lifetime`, so that the import cycle does not fail when the modules load. From now on, the `accounts` tests need the `SiteSettings` table, and a change to `SiteSettings` can break password resets. The request asked for a setting, but not for this dependency. The project already has `core.settings` for module settings: `billing/settings.py:3` registers `billing.trial_days` with `core.settings.register`, and the site settings page shows each registered setting (`admin/views.py:58`). In my judgment, the setting belongs in `core.settings`: this is how the project connects modules to settings, it removes the import cycle, and it needs no migration, because `core.settings` stores each value as a row.

### Implementation details

```diff
+def reset_lifetime():
+    from admin.models import SiteSettings
+    return timedelta(hours=SiteSettings.load().reset_link_hours)
+
 def is_expired(token):
     age = datetime.now() - token.created_at
-    return age > RESET_LIFETIME
+    return age > reset_lifetime()
```

`admin/models.py:20` adds `reset_link_hours = models.PositiveIntegerField(default=1)`, and `admin/migrations/0007_sitesettings_reset_link_hours.py` adds the column. `SiteSettings.load()` reads the single settings row, so each check makes one more database query.
````
