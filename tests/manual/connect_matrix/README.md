# Connect matrix

`run_connect_matrix.sh` drives `socketio_open` against real destinations and
reports the platform error code and the elapsed time for each attempt.

## Why this exists alongside the unit tests

`tests/socketio_win32_ut` mocks `getaddrinfo`, `connect` and `select`, so it can
assert exactly which candidates the connect loop attempts and in which order.
That is the right place for the logic.

What it cannot show is the part that is a *timing* behaviour:

- that a genuinely blackholed address is abandoned when the budget expires,
  rather than running to the OS timeout - about 21 s on Windows;
- that an immediate refusal really does cost nothing from the shared budget;
- that a reachable candidate sitting behind blackholed ones is still reached.

This harness measures those directly, on a real stack.

## Running

```
cmake <repo> -DCMAKE_BUILD_TYPE=Release -Dskip_samples=ON -Duse_openssl=ON -Drun_unittests=OFF
make aziotsharedutil
tests/manual/connect_matrix/run_connect_matrix.sh <build-dir>
```

Needs `gcc`, `python3` and `unshare`. The multi-address cases supply a private
`/etc/hosts` inside a mount namespace, so the host's own `/etc/hosts` is never
written; the script verifies its checksum afterwards and reports any leaked
mount.

Set `BLACKHOLE_PREFIX` if `fd00:c8de:3ffb:9999::` is not inside a range your
host routes. The addresses must *route* and not answer - an address that is
rejected with `ENETUNREACH` returns immediately and does not exercise the
budget at all.

Pick a port that is not intercepted. Behind a corporate proxy, 443 is often
accepted for any address, which makes a blackhole look like an instant success.
The script uses 8081.

## Measured behaviour on this branch

Measured on Ubuntu 24.04, from `09560c1bcb` ("Ask for AI_ADDRCONFIG in the
Berkeley resolver hint"), on two hosts with opposite address-family preference.
Identical results on both.

Single candidate:

```
host=192.0.2.1          port=8081  result=OPEN_ERROR code=110  elapsed_ms=10000
host=203.0.113.1        port=8081  result=OPEN_ERROR code=110  elapsed_ms=10000
host=127.0.0.1          port=9     result=OPEN_ERROR code=111  elapsed_ms=0
host=::1                port=9     result=OPEN_ERROR code=111  elapsed_ms=0
host=::ffff:127.0.0.1   port=9     result=OPEN_ERROR code=111  elapsed_ms=0
```

`110` is `ETIMEDOUT`, `111` is `ECONNREFUSED`. The blackholes stop at exactly
`CONNECT_TIMEOUT_MS`, which matches what `socketio_win32.c` now does - Windows
measured 10002 ms against 21075 ms for a plain blocking connect to the same
address. The bound is at parity across the two adapters.

`::ffff:127.0.0.1` is *accepted* here and reaches the IPv4 stack: against a port
with a listener it returns `OPEN_OK`. Linux leaves `IPV6_V6ONLY` off, whereas
`socketio_win32.c` leaves it at the Windows default and the same literal is
refused with `WSAEADDRNOTAVAIL`. Both are as implemented; the difference is
expected and is worth stating somewhere, because only the Windows side is
currently covered by a test.

Preferred family blackholed, with a reachable IPv4 candidate behind it:

| Blackholed IPv6 candidates | Result | IPv4 attempted |
| --- | --- | --- |
| 1 | `OPEN_OK` after 5000 ms | yes |
| 2 | `OPEN_ERROR` 110 after 10000 ms | no |
| 3 | `OPEN_ERROR` 110 after 10000 ms | no |

One blackholed candidate is capped at `CONNECT_ATTEMPT_TIMEOUT_MS` (5000 ms),
leaving 5000 ms for the IPv4 candidate, which connects. From two onwards the
shared 10 s budget is spent entirely inside the preferred family and the other
family is never attempted.

`socketio_win32.c` holds budget back for a family it has not tried yet -
`CONNECT_FAMILY_RESERVE_MS` together with `untried_family_ahead()` - and
`tests/socketio_win32_ut` covers it in
`socketio_open_reserves_budget_for_untried_address_family`.
`socketio_berkeley.c` has no equivalent, and there is no corresponding Berkeley
unit test.

This matters wherever a host prefers IPv6 and IPv6 is broken while IPv4 works,
which is the common failure mode this feature is meant to survive. A host using
DNS64 is exposed to it too: DNS64 synthesises a AAAA for every IPv4-only
service, so if NAT64 stops forwarding, every synthesised address blackholes at
once and there is usually more than one.

The gap is acknowledged in the commit message of `09560c1bcb`:

> It is not a substitute for iterating the candidate list. An interface can be
> configured while the destination stays unreachable, which is the case the
> Windows family reserve covers and Berkeley still does not.

The measurements above are what that reads like on a real host.

## Note on running the unit tests here

`host_utils_ut` and the other `umock_c` suites do not build with GCC 13: the
vendored `testtools/umock-c/src/umocktypes_charptr.c` trips
`-Werror=stringop-overread`, and umock-c appends `-Werror` itself so a
`-Wno-error` on the command line does not take effect. This is pre-existing and
unrelated to the IPv6 changes, but it does mean the unit suites could not be
run on the validation hosts.
