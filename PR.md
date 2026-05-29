# Fix: missing /usr/sbin/httpd / dangling /bin/httpd after default install

## Summary

- Default v1.0.1 installs (without `--enable-root-access`) ship a dangling
  symlink `/bin/httpd → /bin/busybox2` because `cp /bin/busybox2 /tmp/1/bin/busybox2`
  was incorrectly placed inside the `ENABLE_ROOT_ACCESS` conditional in
  `build.sh`, while the corresponding `ln -sf` runs unconditionally a few
  lines later. As a result the nleapi init script cannot launch httpd and
  the post-install URL-config step fails with `fetch failed`.
- Move the busybox2 copy + chmod out of the conditional so every build ships
  it onto the persistent rootfs. The dropbear/autossh deps stay gated on
  root access.
- Fix a related bug in `httpd.monitrc`: it was watching `/usr/sbin/httpd`
  but the init script launches `${BINDIR}/httpd` (= `/bin/httpd`), so monit
  would never see the running process and would keep restarting it until
  hitting the unmonitor threshold.

## Why

Reported in the wild: after flashing v1.0.1 on a Nest Gen 2 (Display 2.14
/ firmware 4.3.3 / Backplate 2.21), the installer's URL-config step fails
with `Configuration failed: fetch failed`. On the device,
`/etc/init.d/nleapi` and `/var/www/cgi-bin/` are present but
`/usr/sbin/httpd` does not exist; `/bin/httpd` exists as a symlink to
`/bin/busybox2` which is also missing. The known workaround is to drop a
static `busybox-with-httpd` onto the device manually.

This fix removes the need for the workaround.

## Test plan

- [ ] `./docker-build.sh --generation gen2 --yes` (default options) and
      confirm the generated `firmware/builder/deps/root/etc/init.d/rootme`
      contains an unconditional `cp /bin/busybox2 /tmp/1/bin/busybox2`
      between the mtdblock7 mount and the (now empty) ENABLE_ROOT_ACCESS
      block.
- [ ] Same build, extract the embedded initramfs from `firmware/uImage`
      and confirm the rootme inside matches.
- [ ] Flash to a spare Gen 2 thermostat. After the post-install reboot,
      verify on device:
      - `ls -l /bin/busybox2 /bin/httpd` — both present, symlink resolves.
      - `pidof httpd` returns a PID and `cat /proc/$(pidof httpd)/cmdline`
        starts with `/bin/httpd`.
      - `curl -sS http://localhost:8080/cgi-bin/version` returns the
        expected version string.
      - The post-flash installer's URL-config step completes without
        `fetch failed`.
- [ ] Repeat with `--enable-root-access` to confirm no regression in the
      root-access path (busybox2 still gets copied, plus autossh/dropbear
      as before).
- [ ] Confirm monit doesn't keep retrying httpd: tail
      `/var/log/monit.log` for a couple of minutes after boot.
