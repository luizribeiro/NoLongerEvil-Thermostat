# Fix: monit fails to start because `httpd.monitrc` is included twice

## Summary

`firmware/builder/build.sh` appends `include /etc/monit.d/httpd.monitrc` to
the device's `/etc/monitrc`, guarded only by `grep -q "httpd.monitrc"`. The
Nest base firmware's `/etc/monitrc` already contains
`include /etc/monit.d/*.monitrc`, which the grep doesn't recognise as
covering the same file. Result: `httpd.monitrc` gets parsed twice, monit
hits a duplicate-service-name error and exits, and the post-install
URL-config step then fails with `fetch failed` because httpd was never
started.

Widen the guard so it matches either form:

```sh
if ! grep -qE "include /etc/monit\.d/(\*|httpd)\.monitrc" /tmp/1/etc/monitrc; then
  echo "include /etc/monit.d/httpd.monitrc" >> /tmp/1/etc/monitrc
fi
```

On stock 4.3.3 (and presumably any firmware shipping the standard Nest
monitrc) the glob is already present and the append is now skipped, so
monit parses cleanly. On any hypothetical base without the glob, the
explicit include is still added — so behavior is preserved.

## Why

Reproduced on a Nest Gen 2 (Display-2.14 / firmware 4.3.3 /
Backplate-2.21) installed with v1.0.1 + `--enable-root-access`:

```
# /usr/sbin/monit -I -t
/etc/monit.d/httpd.monitrc:8: Error: service name conflict,
   httpd already defined '/usr/sbin/httpd'
```

```
# tail -2 /etc/monitrc
include /etc/monit.d/*.monitrc
include /etc/monit.d/httpd.monitrc
```

After hand-removing the explicit include and running
`/etc/init.d/monit start`, monit stays up and `nleapi`'s
`monit_service_action start httpd` succeeds (which is what the rcS-time
`${INITDIR}/nleapi start` call relies on).

## Test plan

- [ ] `./docker-build.sh --generation gen2 --enable-root-access` (and a
      second build without root access). For each, extract the generated
      rootme bootstrap and confirm it now contains the wider grep guard.
- [ ] Flash to a spare Gen 2. After the post-install reboot:
  - `/usr/sbin/monit -I -t` parses without errors.
  - `pidof monit` returns a PID.
  - `pidof httpd` returns a PID; `cat /proc/$(pidof httpd)/cmdline`
    starts with `/bin/httpd`.
  - `curl -sS http://localhost:8080/cgi-bin/version` returns the version
    string.
  - The desktop installer's URL-config step completes without
    `fetch failed`.

## Not included

There are two adjacent bugs in v1.0.1 that I noticed while investigating
but deliberately left out of this PR (each warrants its own focused
change + test plan; full write-up in `INVESTIGATION.md`):

1. `cp /bin/busybox2 /tmp/1/bin/busybox2` is gated on
   `ENABLE_ROOT_ACCESS`, but the symlink that depends on it is
   unconditional — default installs may ship a dangling `/bin/httpd →
   /bin/busybox2`. Not verified against a real default install; this
   PR's reporter used `--enable-root-access` so the cp ran.
2. `httpd.monitrc` has `matching /usr/sbin/httpd` while the init script
   launches `${BINDIR}/httpd` (= `/bin/httpd`). Moot until (B) is fixed
   and monit actually runs, but worth a follow-up.
