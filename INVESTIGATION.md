# NLE v1.0.1 — missing httpd investigation

## Summary

### Root cause

After installing NLE v1.0.1 on a Nest Gen 2 (Display-2.14 / firmware 4.3.3 / Backplate-2.21), monit fails to start. Running it in the foreground gives:

```
/etc/monit.d/httpd.monitrc:8: Error: service name conflict,
   httpd already defined '/usr/sbin/httpd'
```

The Nest base firmware's `/etc/monitrc` already contains:

```
include /etc/monit.d/*.monitrc
```

`build.sh` (line 609–611) then appends an *explicit* include for the same file:

```sh
if ! grep -q "httpd.monitrc" /tmp/1/etc/monitrc; then
  echo "include /etc/monit.d/httpd.monitrc" >> /tmp/1/etc/monitrc
fi
```

The grep guard only looks for the literal string `httpd.monitrc`; it doesn't recognise that the pre-existing `include /etc/monit.d/*.monitrc` glob already covers it. So `httpd.monitrc` gets parsed twice → duplicate `httpd` service definition → monit refuses to load any config and exits silently. Without monit, `/etc/init.d/nleapi start` (which goes through `monit_service_action start httpd`) is a no-op, httpd never comes up at boot, and the post-install URL-config step fails with `fetch failed`.

### Fix

One-line change in `firmware/builder/build.sh`: widen the grep guard so it matches either the glob include or the explicit include:

```sh
if ! grep -qE "include /etc/monit\.d/(\*|httpd)\.monitrc" /tmp/1/etc/monitrc; then
  echo "include /etc/monit.d/httpd.monitrc" >> /tmp/1/etc/monitrc
fi
```

On the stock 4.3.3 base, the glob include is already present → no append → no duplicate → monit parses and starts. On any hypothetical base without the glob, the explicit include is still added → httpd.monitrc still gets loaded.

### Verification

On a live exploited device (this user's, currently running NLE v1.0.1 with the workaround patches in place):

- `/usr/sbin/monit -V` → "Monit version 5.4" (monit IS installed in the base firmware).
- `/etc/init.d/monit start` returns success but `pidof monit` is empty afterwards.
- `monit -I -t` (foreground, validate config) prints the duplicate-service error above.
- `/etc/monitrc` ends with both `include /etc/monit.d/*.monitrc` and `include /etc/monit.d/httpd.monitrc`.
- `/etc/monit.d/httpd.monitrc` exists exactly once on disk; the duplication is purely an include-then-include-again issue.

After hand-removing the explicit `include /etc/monit.d/httpd.monitrc` line from `/etc/monitrc` and restarting monit, monit stays up.

### Not verified (do before merging)

- **Did not rebuild the firmware end-to-end.** The fix is in the build script that generates the rootme bootstrap; a full Docker build + DFU flash on a spare Gen 2 is the right confirmation.
- **Did not test on a firmware variant other than 4.3.3.** The fix degrades gracefully (still appends the explicit include if no glob is present), so it should be safe across versions, but I only have one device to confirm against.

---

## Other suspected bugs (not fixed in this PR)

Two additional issues turned up while investigating; both are real-looking from a static reading of `build.sh`, but neither appears to actually fire on a `--enable-root-access` install. They're called out here in case upstream wants separate PRs.

### A. `cp /bin/busybox2` is gated on `ENABLE_ROOT_ACCESS`

In v1.0.1's `build.sh`, the line that copies `busybox2` onto the persistent rootfs lives inside the `if [ "$ENABLE_ROOT_ACCESS" = true ]` block:

```sh
if [ "$ENABLE_ROOT_ACCESS" = true ]; then
  cp /bin/busybox2 /tmp/1/bin/busybox2 || true
  cp /bin/autossh /tmp/1/bin/autossh || true
  chmod 777 /tmp/1/bin/busybox2
  ...
fi
```

But the symlink that *depends* on it is unconditional:

```sh
ln -sf /bin/busybox2 /tmp/1/bin/httpd || true
```

In theory, a default install (without `--enable-root-access`) would ship a dangling symlink `/bin/httpd → /bin/busybox2` with no target binary present. Not verified on a real default install — the device used for this investigation was built with `--enable-root-access`, so the cp ran. If the bug is real, fixing it is a separate one-liner (move the cp + chmod out of the conditional).

### C. `matching` path in `httpd.monitrc` doesn't match the launch path

`httpd.monitrc` (shipped in `firmware/builder/deps/`) has:

```
check process httpd matching /usr/sbin/httpd
  start program = "/etc/init.d/nleapi monit_start"
```

But `monit_start` in `deps/nleapi` invokes `${STARTDAEMON} -a "${HTTPDCLIENTAPP}"` where `HTTPDCLIENTAPP=${BINDIR}/httpd`. And the base firmware sets `BINDIR=${ROOTDIR}bin` = `/bin`. So the launched process is `/bin/httpd` (= `/bin/busybox2` via symlink), not `/usr/sbin/httpd`. monit's `matching` would never see the running process. Moot on this device because monit isn't running at all (see B), but if/when B is fixed, this becomes the next bug. One-line fix: change `matching /usr/sbin/httpd` → `matching /bin/httpd`.

---

## Why the user's workaround "works"

The widely-shared community workaround is "drop a static `busybox-armv5l` at `/usr/sbin/httpd`, chmod +x, run `/etc/init.d/nleapi monit_start`". On this device:

- The `/usr/sbin/httpd` binary is a **placebo** — nothing actually invokes that path. `nleapi monit_start` runs `${BINDIR}/httpd` (= `/bin/httpd` → `/bin/busybox2`), bypassing both monit and the wrong-path binary.
- The thing that actually rescues the install is `monit_start` itself, which sidesteps the (dead) monit entirely and launches httpd directly. Once httpd is up long enough for the desktop installer's URL-config POST to land, the device is configured and never needs httpd again.

This explains why the device runs fine indefinitely after install: post-bootstrap, all comms are outbound (transport.put / transport.subscribe to the configured cloud URL), and httpd is dead weight. `pidof httpd` on a long-running NLE device is expected to be empty.
