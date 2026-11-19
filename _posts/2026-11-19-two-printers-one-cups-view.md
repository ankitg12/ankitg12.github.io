---
layout: post
title: "Two Printers, One List: What CUPS Taught Me About Print Servers"
date: 2026-11-19
categories: linux printing
---

A USB printer was connected, detected, and able to print a test page. Yet it did not appear in the browser's print dialog. The dialog instead showed a long list of printers from a remote print server. Restarting the browser did not change that.

The problem was not that Linux could not support two printers. The applications were asking one print server for destinations, and the USB printer was registered with another. The fix was to make the local print server the application-facing entry point, then add a local queue that forwards to the existing remote queue. No shared printers needed to be deleted.

This post describes the method, the protocol details, and the limits of what the tests proved. Names, domains, and account identifiers in the examples are fictional; replace them with your own values. The commands are a recipe, not a transcript of my machine.

## The key distinction: a printer is not a queue

A **physical printer** is a device that puts marks on paper. A **queue** is a software-managed destination that accepts print jobs, applies policy and processing, and sends work to a device or another service. One physical printer can have more than one queue; a queue can also point to a remote print service rather than directly to a device.

On my Linux machine, [CUPS](https://www.cups.org/doc/overview.html) supplied the queue manager. The setup initially looked like this:

```text
Browser / print dialog
       |
       | CUPS client configuration
       v
Remote print server  ---> many remote queues

Local CUPS server    ---> USB printer queue
       ^
       +--- not the server the browser was querying
```

A USB device being present is not enough to put it in the list returned by a different server. A browser restart cannot fix a server-selection mismatch: it may refresh the list, but it still queries the configured server.

The useful diagnostic is to compare the servers explicitly. [`lpstat`](https://www.cups.org/doc/man-lpstat.html) can show the queues and their device URIs:

```sh
# Replace print.example.test with your real print-server name.
lpstat -h print.example.test -e
lpstat -h localhost -e
lpstat -h localhost -v
```

`-e` lists available destinations. `-v` shows where a queue sends jobs. On the machine in this case, the remote server listed many destinations, while the local server had the USB queue. I did not delete anything from the remote server.

## The topology that solved it

I kept the existing remote print service and introduced a local forwarding queue:

```text
Browser / print dialog
       |
       v
Local CUPS server
       |
       +--> HOME_USB   -- USB backend --> physical printer
       |
       +--> REMOTE_PRINT -- IPP --> remote print server
                                      |
                                      +--> existing remote queue
                                           and its release workflow
```

The local server is the **aggregation point**. It presents a small list to local applications. The remote server remains responsible for its own queues, policies, and downstream delivery. This is local curation, not deletion or reconfiguration of a shared service.

In my setup the CUPS destination list then contained the USB queue and the forwarding queue. The forwarding queue stayed the default. I did not independently verify the changed list inside the browser.

This architecture is not a universal client-side filter. Local discovery services can add other printers, and other user accounts or applications can have different configurations. Inspect the actual local list rather than assuming it will always contain exactly two entries.

## Prior art: an established pattern, not a new invention

Others have run into crowded destination lists and the distinction between local and remote servers. In [CUPS issue 5011](https://github.com/apple/cups/issues/5011), opened in 2017, developers discussed filtering large numbers of destinations discovered through DNS-based service discovery. That issue concerns **discovered destinations**, not exactly the same mechanism as a client configured to query a remote server. It is nevertheless useful prior art for the broader problem: printer discovery and printer selection are different concerns. A separate [CUPS issue from 2012](https://github.com/apple/cups/issues/4231) discusses applications and `ServerName` when redirecting printing to a remote server.

Manual queue creation is an established CUPS capability; its [network printer documentation](https://www.cups.org/doc/network.html) explains configuring network destinations. The contribution here is the specific diagnosis and the verified combination: one local view, one explicit forwarding queue, preserved job identity, and no deletion of remote queues.

| Approach | How it works | When it fits |
|---|---|---|
| Explicit local forwarding queue | Create a local queue for one known remote IPP endpoint. | A small, stable set of destinations; each local queue needs maintenance if its remote endpoint changes. |
| [cups-browsed](https://github.com/OpenPrinting/cups-browsed) | A separate service discovers or polls supported remote printers and can create local queues; its [configuration](https://manpages.debian.org/bookworm/cups-browsed/cups-browsed.conf.5.en.html) includes `BrowsePoll` and `BrowseFilter`. | Larger or changing environments where an administrator wants managed discovery and filtering. Behavior and supported directives depend on the installed version. |
| Point clients directly at the remote server | Configure `ServerName` to use the remote CUPS server. | A simple remote-only setup; local USB queues on a separate local server are not automatically merged into that remote server's queue list. |

The automatic approach is not always preferable. Discovery can create additional destinations, and configuration of a system service can affect other users. For this one stable remote queue, the explicit queue kept the change small. That is a choice for this situation, not a claim that it is universally best.

## What IPP does—and does not—specify

The forwarding hop uses the [Internet Printing Protocol (IPP)](https://www.rfc-editor.org/info/std92/). IPP/1.1 is Internet Standard 92, published as two documents:

| Specification | Responsibility |
|---|---|
| [RFC 8010](https://www.rfc-editor.org/rfc/rfc8010.html) | Encoding and transport: IPP messages and attributes in `application/ipp`, carried over HTTP or HTTPS. |
| [RFC 8011](https://www.rfc-editor.org/rfc/rfc8011.html) | Model and semantics: Printer and Job objects, operations, attributes, and state. |

An IPP printer is a logical service endpoint. It need not be the physical printer. For example, a remote CUPS queue can expose an IPP endpoint while its own backend sends the work elsewhere. A local queue forwarding to that endpoint adds another administrative hop; it does not turn the local machine into the remote server.

A simplified job path looks like this:

1. The application submits a document and job attributes to local CUPS.
2. Local CUPS processes the document according to the queue's driver and options.
3. Its IPP backend submits a job to the remote queue.
4. The remote service applies its own queue and release policies.
5. A printer eventually marks paper, if the downstream path and release succeed.

At each hop, "accepted" or "completed" refers to that hop's state, not automatic proof that paper came out. A local forwarding job can complete after the remote service accepts it. A pull-print system can still require a separate release at the device. Physical output is the final check.

### A stopped queue can still accept work

IPP exposes `printer-state` separately from `printer-is-accepting-jobs`. The former describes whether a Printer is idle, processing, or stopped. The latter says whether it accepts new jobs. They answer different questions: a stopped queue can accept and retain jobs without processing them.

In this case, the remote queue reported disabled but accepting. A test submitted through the local forwarding queue appeared on the remote queue under the intended account. I then canceled that test. This confirmed delivery to the remote queue, **not** release or printing at a physical device. A stopped queue is an operational condition to investigate with its administrator, not a reason to claim end-to-end success.

### Identity travels with the job

For a pull-print workflow, queue visibility alone is not sufficient: the remote service must associate the job with the right user. IPP defines `requesting-user-name` and `job-originating-user-name`; servers also apply their own authentication and authorization policies. A name in a request is not, by itself, proof of identity.

The CUPS IPP backend receives the local job's user and uses it when it creates the remote request. In the [CUPS 2.4.7 IPP backend source](https://github.com/OpenPrinting/cups/blob/v2.4.7/backend/ipp.c), the job-creation path passes the backend's user argument to `new_request`. That is implementation evidence for this version, not a guarantee that every print server or authentication scheme maps identities in the same way.

I configured the client identity to match the remote workflow and checked the remote job owner. If your server requires Kerberos, TLS client certificates, different account mapping, or another policy, follow its administrator's guidance. Do not assume that changing a displayed username grants authorization.

## A practical setup

The steps below are illustrative. Do not paste the placeholders as real hostnames or usernames. First record your existing settings and make sure you know how to restore them. Printer administration can need local administrator rights, and changing a shared queue can affect other people. The aim here is to create or modify **local** queues only.

### 1. Find the two endpoints

```sh
# Remote queue and its server-side device URI:
lpstat -h print.example.test -p REMOTE_PRINT -v REMOTE_PRINT

# Local queues and USB device URI:
lpstat -h localhost -e
lpstat -h localhost -v
```

An existing remote queue's device URI may be a different protocol or a different host from the remote queue's IPP URI. To forward through the existing service, use the **remote queue's supported IPP URI**, not an internal backend URI copied blindly from `lpstat -v`. The remote administrator or `printer-uri-supported` attribute can identify the endpoint.

For a USB device, [USB-IF's printer class specification](https://usb.org/sites/default/files/usbprint11a021811.pdf) defines the USB printer interface; CUPS still needs a suitable queue and, for non-driverless devices, a compatible driver. In my case the printer had a model-specific [brlaser](https://github.com/pdewacht/brlaser) driver. A driverless [IPP Everywhere](https://www.pwg.org/ipp/everywhere.html) network printer is a different case: don't assume a USB-only device supports it.

### 2. Create a local forwarding queue

Choose an appropriate driver or a supported driverless configuration for your actual remote endpoint. The following shows the relationship, not a universal driver prescription:

```sh
# Administrative operation: this changes the LOCAL CUPS server only.
# Replace <model> with an appropriate locally available model.
lpadmin -h /run/cups/cups.sock \
  -p REMOTE_PRINT -E \
  -v ipp://print.example.test:631/printers/REMOTE_PRINT \
  -m '<model>'
```

[`lpadmin`](https://www.cups.org/doc/man-lpadmin.html) documents `-p`, `-v`, `-m`, and `-E`. The socket path is system-specific. Use `lpinfo -m` to inspect locally available models, and choose one that is appropriate for the remote service's document formats. Traditional model/PPD-based drivers are deprecated by CUPS; `-m everywhere` is appropriate only when the destination actually supports the required driverless behavior. Verify rather than assuming.

Keep a local forwarding queue private unless sharing is intended:

```sh
lpadmin -h /run/cups/cups.sock -p REMOTE_PRINT \
  -o printer-is-shared=false
```

On my system, local administrator authentication worked through the CUPS Unix socket. A TCP connection to `localhost` had prompted separately. This is a property of the local CUPS policy and session, not a promise that socket access bypasses authentication. Do not weaken permissions or change a shared server to make setup convenient.

If the queue is not enabled or accepting, inspect it before changing it:

```sh
lpstat -h localhost -p REMOTE_PRINT -a REMOTE_PRINT
```

The local queue's state and the remote queue's state are separate. Enabling a local queue does not repair a stopped remote service.

### 3. Select the local server for applications

The per-user [CUPS client configuration](https://www.cups.org/doc/man-client.conf.html) controls which server CUPS-aware applications contact. On Linux, the relevant setting can be placed in `~/.cups/client.conf`:

```text
ServerName localhost
User user@example.test
```

Use your actual account identity, if the remote service requires one. This is a user-scoped choice, but it affects applications using that configuration. Some sandboxed applications have a separate home or their own printing integration; check them individually. A browser may need its print dialog reopened or a restart to pick up changes. That behavior depends on the browser and was not verified independently for every browser in this case.

Set the desired default deliberately, and check both destinations:

```sh
lpoptions -d REMOTE_PRINT
lpstat -d
lpstat -e
lpstat -p REMOTE_PRINT -p HOME_USB
```

Do not change a working default until you have checked where it sends jobs and how to restore it. The original remote route can also be used explicitly with `lp -h`, subject to that server's policies.

## Test each boundary, not just the final dialog

The validation I used separated four questions:

| Boundary | Check | What it proves—and what it does not |
|---|---|---|
| USB detection | `lsusb`; inspect the local queue | The computer detects a device and CUPS has a destination; it does not prove output. |
| Local print path | Submit a test page to the USB queue | CUPS reported completion; inspect the page for actual physical output. |
| Forwarding | Submit a controlled test to the local forwarding queue; inspect the remote queue | A remote job appeared with the expected owner. It does not prove badge release or paper output. |
| Application view | Inspect `lpstat -e`, then the browser dialog | CUPS clients see the selected server's destinations. The command alone does not prove what a particular browser displays. |

[`ipptool`](https://www.cups.org/doc/man-ipptool.html) can query protocol attributes without sending a document. `lpstat -p -a` is a useful first check, but for an IPP investigation, compare the actual `printer-state`, `printer-state-reasons`, `printer-is-accepting-jobs`, and job-state attributes at each endpoint.

A test sent to a real shared service is still a real job. Use a document you are authorized to send, avoid sensitive content, and cancel only your own test if you do not intend to release it. A remote job ID is stronger evidence of forwarding than a local "completed" status alone.

## Security and trade-offs

- **An aggregation point adds a hop.** Local CUPS and the network must both work before forwarding can succeed.
- **Identity is part of correctness.** A job can reach the remote server yet be unavailable to the intended user if identity mapping or authentication differs.
- **`ipp://` is not a confidentiality claim.** IPP can run over HTTP or HTTPS; use the security and transport settings required by the service. Do not publish real queue URIs, credentials, or internal hostnames.
- **A CUPS queue is not a security boundary by itself.** Consult the [CUPS server security documentation](https://www.cups.org/doc/security.html) and your administrator before changing sharing or authorization.
- **A local driver can change job handling.** Check media, duplex, color, and document formats at both hops. Local defaults are not proof that the remote service will honor every option.
- **The shared service remains shared.** If its queue is stopped or its release system fails, a local forwarding queue cannot fix it.

## What I did not verify

I verified that the local USB job completed and that a forwarding test reached the remote queue with the expected owner. I did not verify physical release of the forwarded job; the remote queue was disabled, and the test was canceled. I also did not independently verify the destination list inside every browser after the configuration change. Reopen the print dialog and inspect the actual list; if it is stale, restart that browser, subject to its behavior and unsaved work.

The general lesson is that a long printer list can be a **server-selection problem**, not a printer-count limitation. Once the application-facing server and each downstream queue are explicit, the solution is a small routing change—not a cleanup of someone else's print server.

## Source

- [Internet Standard 92: IPP/1.1](https://www.rfc-editor.org/info/std92/), including RFC 8010 and RFC 8011.
- [CUPS client configuration](https://www.cups.org/doc/man-client.conf.html), [printer administration](https://www.cups.org/doc/man-lpadmin.html), and [server security](https://www.cups.org/doc/security.html).
- [CUPS 2.4.7 IPP backend](https://github.com/OpenPrinting/cups/blob/v2.4.7/backend/ipp.c).
- [CUPS issue 5011: filtering discovered destinations](https://github.com/apple/cups/issues/5011), [issue 4231: remote `ServerName`](https://github.com/apple/cups/issues/4231), and [cups-browsed configuration](https://manpages.debian.org/bookworm/cups-browsed/cups-browsed.conf.5.en.html).
- [USB printer class specification](https://usb.org/sites/default/files/usbprint11a021811.pdf) and [IPP Everywhere](https://www.pwg.org/ipp/everywhere.html).
