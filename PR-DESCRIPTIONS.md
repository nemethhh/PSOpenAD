# PR descriptions

One section per PR, each self-contained: its own setup, reproduction, results and
cleanup, so a section can be pasted as a PR body without reference to the others.
Every actual/expected block is captured output from a real run.

## Targets

All six PRs target **`jborean93/PSOpenAD` `main`**. None targets a branch in this
fork, and none depends on another being merged first: each branch's merge base
with `upstream/main` is `510de94`, which is the current upstream head, so all six
apply to it as they stand.

The one exception was `fix/trace-send-buffer`, which was branched off
`fix/large-request-flush` because both fix defects in the same method and share
the test scaffolding. Rather than chain one PR onto another, they are submitted as
a single PR with two commits: **`fix/pipeline-write-data`**. The two older branches
are superseded by it.

| # | Branch | Kind | Setup needed |
|---|--------|------|--------------|
| 1 | `fix/group-scope-bits` | bug | none |
| 2 | `fix/binary-attribute-values` | bug | one contact |
| 3 | `fix/pipeline-write-data` | two bugs, two commits | 1600 contacts (one of the two) |
| 4 | `feat/ranged-retrieval` | bug, needs new code | 1600 contacts + a group |
| 5 | `feat/sd-flags-control` | feature | one contact |
| 6 | `fix/principal-without-sid` | bug, **breaking change** | one contact, one group |

## The CHANGELOG overlap

Every PR adds its entry to the same list under `## v0.8.0 - TBD`, because that is
this project's convention: upstream PRs #91, #94, #100, #101 and #104 each carried
their own `CHANGELOG.md` line. The entries cannot be made conflict-free, since git
sees concurrent inserts at one position as an add/add conflict.

So the first PR merges cleanly and each later one needs a one-line rebase. The
resolution is always the same - **keep both lines** - and nothing else in these
branches overlaps:

| File | Touched by |
|------|-----------|
| `CHANGELOG.md` | all six |
| `src/PSOpenAD/OpenAD.cs` | `fix/group-scope-bits`, `fix/principal-without-sid` (different methods) |
| `src/PSOpenAD.Module/Commands/GetOpenAD.cs` | `feat/ranged-retrieval`, `feat/sd-flags-control` (different regions) |
| everything else | one branch each |

The order in the table above is the suggested merge order: no-setup fixes first,
the breaking change last so it can be held for a version bump. Any order works;
only the rebases differ.

## Validation

All reproductions were validated against Windows Server AD at forest functional
level 2016 with the default `MaxValRange` of 1500, and cross-checked against the
RSAT `ActiveDirectory` module on the DC. Unit tests run with
`dotnet test --project tests/units/PSOpenADTests`; on a host whose only runtime is
newer than the `net8.0` target, export `DOTNET_ROLL_FORWARD=Major` first, as
`tools/lib.sh` does in CI, or the run reports "Zero tests ran".

---

## fix/group-scope-bits

### Problem

`GroupScope` is `Universal` for every security group. `OpenADGroup` compares the
whole `groupType` value against `GroupType.Global` and `GroupType.DomainLocal`,
but a security group also carries `IsSecurity` (`0x80000000`), and the builtin
groups carry `System` (`0x1`) as well, so no comparison matches and every group
falls through to `Universal`. Only a distribution group, which carries no extra
bits, was reported correctly.

### Setup

None. The reproduction uses groups every domain already has.

### Steps to reproduce

```powershell
$session = New-OpenADSession -ComputerName dc.example.com -AuthType Negotiate

foreach ($n in 'Domain Admins', 'Administrators', 'Enterprise Admins') {
    $g = Get-OpenADGroup -Session $session -Identity $n -Property groupType
    '{0,-20} groupType=0x{1:X8}  GroupScope={2}' -f $g.Name, [uint32]$g.GroupType, $g.GroupScope
}
```

### Actual result

```
Domain Admins        groupType=0x80000002  GroupScope=Universal
Administrators       groupType=0x80000005  GroupScope=Universal
Enterprise Admins    groupType=0x80000008  GroupScope=Universal
```

### Expected result

```
Domain Admins        groupType=0x80000002  GroupScope=Global
Administrators       groupType=0x80000005  GroupScope=DomainLocal
Enterprise Admins    groupType=0x80000008  GroupScope=Universal
```

The same values from the `ActiveDirectory` module, which is where the expected
column comes from:

```powershell
Get-ADGroup -Identity 'Domain Admins' | Select-Object Name, GroupScope
# Domain Admins  Global
```

Nothing here is AD specific: on the Samba container the test suite targets, all 41
groups reported `Universal` before the fix and 26 `DomainLocal` / 12 `Global` / 3
`Universal` after it, which is why this is covered by an integration test as well
as unit tests.

### Cleanup

None.

### Change

`src/PSOpenAD/OpenAD.cs` masks the scope bits before comparing.

### Tests

`tests/units/PSOpenADTests/OpenADGroupTests.cs`: security global, security domain
local (with the `System` bit), security universal, and a distribution group as a
regression guard.

`tests/Get-OpenADObject.Tests.ps1` adds "Reports the scope of a security group",
which asserts the three scopes against groups every directory has, so the value a
real server produces is covered and not just a constructed object. It fails on an
unpatched module with `Expected Global, but got Universal`.

The existing suite only asserted that a `GroupScope` property is present, never
its value, which is why this went unnoticed.

---

## fix/binary-attribute-values

### Problem

A value read from `Get-OpenAD*` comes back wrapped in a `PSObject`. Feeding it
straight back into `-Add`/`-Replace` writes the string form of the byte array
instead of the bytes, so the attribute is silently corrupted or the server rejects
the value.

### Setup

```powershell
$session = New-OpenADSession -ComputerName dc.example.com -AuthType Negotiate
$root = 'DC=example,DC=com'
$ou = "OU=psopenad-pr-binary,$root"

New-OpenADObject -Session $session -Name psopenad-pr-binary -Type organizationalUnit -Path $root
New-OpenADObject -Session $session -Name pr-contact -Type contact -Path $ou
```

### Steps to reproduce

```powershell
$dn = "CN=pr-contact,$ou"
Set-OpenADObject -Session $session -Identity $dn -Replace @{ thumbnailPhoto = [byte[]]@(1, 2, 3, 4) }

# read it back and write the value the module just handed us
$read = (Get-OpenADObject -Session $session -Identity $dn -Property thumbnailPhoto).ThumbnailPhoto
Set-OpenADObject -Session $session -Identity $dn -Replace @{ thumbnailPhoto = $read }

$again = (Get-OpenADObject -Session $session -Identity $dn -Property thumbnailPhoto).ThumbnailPhoto
"thumbnailPhoto after feeding the read value back: $([Convert]::ToHexString($again))"
```

### Actual result

```
thumbnailPhoto after feeding the read value back: 31203220332034
```

`31 20 32 20 33 20 34` is the ASCII text `1 2 3 4`: the array was stringified.

### Expected result

```
thumbnailPhoto after feeding the read value back: 01020304
```

The same round trip through the `ActiveDirectory` module keeps the bytes, which is
the behaviour being matched:

```powershell
Set-ADObject -Identity $dn -Replace @{ thumbnailPhoto = [byte[]]@(1,2,3,4) }
$v = (Get-ADObject -Identity $dn -Properties thumbnailPhoto).thumbnailPhoto
Set-ADObject -Identity $dn -Replace @{ thumbnailPhoto = $v }
[BitConverter]::ToString((Get-ADObject -Identity $dn -Properties thumbnailPhoto).thumbnailPhoto)
# 01-02-03-04
```

### Cleanup

```powershell
Get-OpenADObject -Session $session -SearchBase $ou -LDAPFilter '(objectClass=*)' |
    Sort-Object -Property { $_.DistinguishedName.Length } -Descending |
    Remove-OpenADObject -Session $session
```

### Change

`src/PSOpenAD/Schema.cs`: `ConvertToRawAttributeValue` and
`ConvertToRawAttributeCollection` unwrap a `PSObject` before dispatching on the
value's type.

### Tests

`tests/units/PSOpenADTests/SchemaTests.cs` for the wrapped byte array and wrapped
string cases, plus a Pester regression in `tests/Set-OpenADObject.Tests.ps1`
("Round-trips a value read back through -Replace") that fails on the unpatched
module with `31203220332034`.

---

## fix/pipeline-write-data

Two defects in `PipelineLDAPSession.WriteData`, one commit each. They are in one
PR because they are in the same six-line method and the second's test needs the
test scaffolding the first adds.

### Problem 1: a large request fails while the server still applies it

`WriteData` calls `GetResult()` on the `ValueTask` returned by
`PipeWriter.FlushAsync()`. Once a request is larger than the pipe's pause
threshold (64 KiB by default) that flush is still pending, and `GetResult()` on an
incomplete `ValueTask` throws instead of waiting.

The failure is worse than a spurious error: the bytes are already in the pipe, so
the sender still delivers the request and **the server applies it** while the
cmdlet reports failure.

#### Setup

```powershell
$session = New-OpenADSession -ComputerName dc.example.com -AuthType Negotiate
$root = 'DC=example,DC=com'
$ou = "OU=psopenad-pr-flush,$root"

New-OpenADObject -Session $session -Name psopenad-pr-flush -Type organizationalUnit -Path $root

# 1600 members is enough for the add request to pass the pipe's 64 KiB threshold
0..1599 | ForEach-Object {
    New-OpenADObject -Session $session -Name ('pr-c{0:d4}' -f $_) -Type contact -Path $ou
}
```

#### Steps to reproduce

```powershell
$members = 0..1599 | ForEach-Object { "CN=pr-c{0:d4},$ou" -f $_ }

New-OpenADObject -Session $session -Name pr-flushgroup -Type group -Path $ou -OtherAttributes @{
    sAMAccountName = 'pr-flushgroup'
    member         = $members
}

# then look for the object the command said it failed to create
Get-ADGroup -Identity "CN=pr-flushgroup,$ou" -Properties member |
    ForEach-Object { "member count: " + @($_.member).Count }
```

The check uses `Get-ADGroup` on purpose: counting 1600 members with an unpatched
PSOpenAD reports 0, because a group over `MaxValRange` is subject to the ranged
retrieval issue as well.

#### Actual result

```
New-OpenADObject with 1600 members: ERROR -> Can't GetResult unless awaiter is completed.
```

and the group exists anyway, fully populated:

```
member count: 1600
```

#### Expected result

```
New-OpenADObject with 1600 members: OK
member count: 1600
```

#### Cleanup

```powershell
Get-OpenADObject -Session $session -SearchBase $ou -LDAPFilter '(objectClass=*)' |
    Sort-Object -Property { $_.DistinguishedName.Length } -Descending |
    Remove-OpenADObject -Session $session
```

### Problem 2: -TracePath logs the buffer, not the request

`WriteData` calls `TraceMsg` on the memory it has just rented from the pipe,
*before* `Encode` writes the request into it, and passes the whole span rather
than the encoded length. Every `SEND` line is the recycled contents of a pooled
buffer, padded to its full size, so the log is useless for diagnosing what the
module sent.

#### Setup

None.

#### Steps to reproduce

```powershell
$trace = Join-Path ([IO.Path]::GetTempPath()) 'psopenad-trace.log'
if (Test-Path $trace) { Remove-Item $trace -Force }

$so = New-OpenADSessionOption -TracePath $trace
$s = New-OpenADSession -ComputerName dc.example.com -AuthType Negotiate -SessionOption $so
Get-OpenADRootDSE -Session $s | Out-Null
$s | Remove-OpenADSession

$sends = @(Get-Content $trace | Where-Object { $_ -like 'SEND: *' })
"SEND frames logged: $($sends.Count)"
"frame sizes       : $(($sends | ForEach-Object { [Convert]::FromBase64String($_.Substring(6)).Length }) -join ', ')"
$first = [Convert]::FromBase64String($sends[0].Substring(6))
'first frame starts with 0x{0:X2} (0x30 = LDAPMessage SEQUENCE)' -f $first[0]
```

#### Actual result

```
SEND frames logged: 4
frame sizes       : 4096, 4096, 4096, 4096
first frame starts with 0x11 (0x30 = LDAPMessage SEQUENCE)
```

Every frame is exactly the rented buffer size and none is an LDAP message.
Decoding them shows unrelated data: in one capture a `SEND` frame held the DNS
query from the connection setup.

#### Expected result

```
SEND frames logged: 4
frame sizes       : 1615, 114, 148, 492
first frame starts with 0x30 (0x30 = LDAPMessage SEQUENCE)
```

#### Cleanup

```powershell
Remove-Item (Join-Path ([IO.Path]::GetTempPath()) 'psopenad-trace.log') -Force
```

### Change

`src/PSOpenAD.Module/PipelineLDAPSession.cs`: a pending flush is handed to a
`Task`, which does wait, keeping the synchronous path for a flush that has already
completed; and tracing happens after `Encode`, covering only the bytes written.

### Tests

`tests/units/PSOpenADTests/PipelineLDAPSessionTests.cs`:

- a request larger than the pause threshold is written with nothing draining the
  pipe yet, then drained, and the write must complete. Fails on the unpatched
  module with `InvalidOperationException: Can't GetResult unless awaiter is
  completed.` from `System.IO.Pipelines.ThrowHelper`.
- a known request is written to a session with a log writer attached, and the
  logged `SEND` frame must decode to exactly the encoded bytes.

Both fixes are also covered through a session, against the Samba container the
suite targets:

- `tests/Set-OpenADObject.Tests.ps1`, "Sets a value larger than the outgoing pipe
  threshold", writes a 128 KiB attribute value. A single value is enough - the
  threshold is 64 KiB - so no large group is needed. Fails on an unpatched module
  with `Can't GetResult unless awaiter is completed.`.
- `tests/OpenADSession.Tests.ps1`, "Logs the request that was sent", asserts every
  `SEND` line in a `-TracePath` log starts with `0x30`, the BER SEQUENCE an
  LDAPMessage begins with. The existing trace test only checks that the file
  grows, which a log of uninitialised buffers does too. Fails on an unpatched
  module with `Expected 48, but got 39`.

The test project gains a reference to `PSOpenAD.Module`, and that project makes
its internals visible to the tests, so the session can be exercised directly.

---

## feat/ranged-retrieval

### Problem

Active Directory truncates a multivalued attribute at `MaxValRange` (1500 by
default) and renames it in the response: `member` comes back as
`member;range=0-1499`. The module matched on the plain name, so the values were
dropped and a spurious `Member;range=0-1499` property appeared on the output
object instead. A group over the limit looked empty.

### Setup

```powershell
$session = New-OpenADSession -ComputerName dc.example.com -AuthType Negotiate
$root = 'DC=example,DC=com'
$ou = "OU=psopenad-pr-ranged,$root"

New-OpenADObject -Session $session -Name psopenad-pr-ranged -Type organizationalUnit -Path $root

# 1600 members: above the default MaxValRange of 1500
0..1599 | ForEach-Object {
    New-OpenADObject -Session $session -Name ('pr-c{0:d4}' -f $_) -Type contact -Path $ou
}
$members = 0..1599 | ForEach-Object { "CN=pr-c{0:d4},$ou" -f $_ }

# added in chunks so the group can also be built on an unpatched module, where a
# single 1600 value add trips the pipe flush issue
New-OpenADObject -Session $session -Name pr-biggroup -Type group -Path $ou -OtherAttributes @{
    sAMAccountName = 'pr-biggroup'
}
for ($i = 0; $i -lt $members.Count; $i += 200) {
    Set-OpenADObject -Session $session -Identity "CN=pr-biggroup,$ou" -Add @{
        member = $members[$i..([Math]::Min($i + 199, $members.Count - 1))]
    }
}
```

### Steps to reproduce

```powershell
$g = Get-OpenADGroup -Session $session -Identity pr-biggroup -Property member
"member count     : $(@($g.Member).Count)"
"member properties: $(($g.PSObject.Properties.Name | Where-Object { $_ -like 'member*' }) -join ', ')"
```

### Actual result

```
member count     : 0
member properties: Member, Member;range=0-1499
```

### Expected result

```
member count     : 1600
member properties: Member
```

Matching the `ActiveDirectory` module, which pages the attribute for you:

```powershell
@((Get-ADGroup -Identity pr-biggroup -Properties member).member).Count
# 1600
```

### Cleanup

```powershell
Get-OpenADObject -Session $session -SearchBase $ou -LDAPFilter '(objectClass=*)' |
    Sort-Object -Property { $_.DistinguishedName.Length } -Descending |
    Remove-OpenADObject -Session $session
```

### Change

`src/PSOpenAD/RangedAttribute.cs` parses and builds the range option.
`src/PSOpenAD/RangedAttributeAccumulator.cs` owns the paging decisions: what to
request next, recognising the final page (`range=1500-*`), and a 1000 page guard
against a server that never terminates the range.
`src/PSOpenAD.Module/Commands/GetOpenAD.cs` performs each follow-up search, feeds
the result back, and replaces the attribute with the complete value set under its
plain name. A warning is emitted if paging stops on the guard rather than on a
final page.

`-Property` still rejects a raw ranged name such as `member;range=0-1` exactly as
before this branch: accepting it and then paging the whole attribute anyway would
contradict itself.

### Tests

`RangedAttributeTests.cs` (parsing, including `userCertificate;binary`, which must
not be treated as a range) and `RangedAttributeAccumulatorTests.cs` (two page
completion, a page with no values, a missing page, and the MaxPages guard).

There is no integration test, because the Samba container the suite targets cannot
produce the response this fixes: it returns all 1600 values with the attribute's
plain name and never a `range=` one. That was checked both for a linked attribute
(`member` on a 1600 member group) and a non-linked one (1600 values on a
multivalued string attribute), on a patched and an unpatched module alike. The
paging decisions are therefore covered by unit tests, and the end to end behaviour
was validated against AD as shown above.

---

## feat/sd-flags-control

### Problem

There is no way to say which components of `nTSecurityDescriptor` a request
covers. Two consequences:

- A read returns whatever the server is willing to return. For a caller without
  the privilege to read the SACL, AD withholds the **whole attribute**, because an
  unmasked read implicitly asks for the SACL too.
- A write sends the whole descriptor. Writing back a descriptor carrying an owner,
  group or SACL the caller cannot write is refused, even when the caller only
  meant to change the DACL.

`LDAP_SERVER_SD_FLAGS` (`1.2.840.113556.1.4.801`) is how a client states which
components it means.

### Setup

```powershell
$session = New-OpenADSession -ComputerName dc.example.com -AuthType Negotiate
$root = 'DC=example,DC=com'
$ou = "OU=psopenad-pr-sdflags,$root"

New-OpenADObject -Session $session -Name psopenad-pr-sdflags -Type organizationalUnit -Path $root
New-OpenADObject -Session $session -Name pr-contact -Type contact -Path $ou
```

### Steps to reproduce

```powershell
$dn = "CN=pr-contact,$ou"
foreach ($mask in 'Dacl', 'Owner') {
    $sd = (Get-OpenADObject -Session $session -Identity $dn `
            -Property nTSecurityDescriptor -SecurityMask $mask).NTSecurityDescriptor
    '-SecurityMask {0,-6} Owner={1,-5} Group={2,-5} Dacl={3,-5} Sacl={4}' -f $mask,
        $(if ($sd.Owner) { 'set' } else { 'null' }), $(if ($sd.Group) { 'set' } else { 'null' }),
        $(if ($sd.DiscretionaryAcl) { $sd.DiscretionaryAcl.Count } else { 'null' }),
        $(if ($sd.SystemAcl) { 'set' } else { 'null' })
}
```

### Actual result

```
-SecurityMask parameter: not available on this build
```

### Expected result

Each mask returns only the components it names:

```
-SecurityMask Dacl   Owner=null  Group=null  Dacl=29    Sacl=null
-SecurityMask Owner  Owner=set   Group=null  Dacl=null  Sacl=null
```

and a masked write sends only those components. Measured on the wire with
`-TracePath`, writing back a descriptor taken from an unmasked read:

```
before the trim: 1304 bytes, control=0x8C17, SACL at offset 76, owner and group present
after the trim:  1128 bytes, control=0x8404, owner/group/SACL offsets all 0
```

### Cleanup

```powershell
Get-OpenADObject -Session $session -SearchBase $ou -LDAPFilter '(objectClass=*)' |
    Sort-Object -Property { $_.DistinguishedName.Length } -Descending |
    Remove-OpenADObject -Session $session
```

### Change

`src/PSOpenAD/LDAP/Control.cs` adds `SecurityDescriptorFlags` and
`SecurityDescriptorFlagsControl`. `GetOpenAD.cs` and `SetOpenAD.cs` add
`-SecurityMask` and attach the control to the search, the modify, and the
`-PassThru` read back. `SecurityDescriptor.cs` gains a copy constructor limiting a
descriptor to the masked components and clearing the control flags of the ones it
drops; `Set-OpenADObject` trims a descriptor value through it.

The trim is stricter than the servers are: a DACL only read returns `0x8C04` from
AD, which keeps `SystemAclAutoInherited` with no SACL present, and `0x8404` from
Samba. Both accept either form.

### Tests

`SecurityDescriptorFlagsControlTests.cs` pins the control's BER encoding against a
known-good byte string. `SecurityDescriptorTrimTests.cs` covers the trim and the
flag clearing. Eight Pester integration tests in `Get-OpenADObject.Tests.ps1` and
`Set-OpenADObject.Tests.ps1` run against the Samba container: the read tests
assert each mask returns only its own components, and the write test adds an ACE
to a descriptor, writes it under a mask covering the owner alone, and asserts the
ACE did not land, then writes the same descriptor with no mask and asserts it
does, so the assertion is about the control rather than the payload.

Verified by dropping the control from `Get`/`Set` while keeping the parameter: six
of the eight fail; the two that pass are the deliberate no-mask control cases.

---

## fix/principal-without-sid

### Problem

`OpenADPrincipal` falls back to `new SecurityIdentifier("")` when an object has no
`objectSid`, and that constructor throws. Any command returning such a principal
fails; `Get-OpenADGroupMember` returns nothing at all for a group containing a
contact.

### Setup

```powershell
$session = New-OpenADSession -ComputerName dc.example.com -AuthType Negotiate
$root = 'DC=example,DC=com'
$ou = "OU=psopenad-pr-sid,$root"

New-OpenADObject -Session $session -Name psopenad-pr-sid -Type organizationalUnit -Path $root
New-OpenADObject -Session $session -Name pr-contact -Type contact -Path $ou
New-OpenADObject -Session $session -Name pr-group -Type group -Path $ou -OtherAttributes @{
    sAMAccountName = 'pr-group'
    member         = @("CN=pr-contact,$ou")
}
```

A contact is the point: it is a valid group member and has no `objectSid`.

### Steps to reproduce

```powershell
Get-OpenADGroupMember -Session $session -Identity pr-group
```

### Actual result

```
members returned: 0   errors: 1
error : System.Management.Automation.CmdletInvocationException: sid
```

### Expected result

```
members returned: 1   errors: 0
member: pr-contact  class=contact  SID=<null>
```

That the member genuinely has no SID, per the `ActiveDirectory` module:

```powershell
$m = (Get-ADGroup -Identity pr-group -Properties member).member
Get-ADObject -Identity $m[0] -Properties objectSid, objectClass |
    Select-Object objectClass, objectSid
# contact   (objectSid empty)
```

### Cleanup

```powershell
Get-OpenADObject -Session $session -SearchBase $ou -LDAPFilter '(objectClass=*)' |
    Sort-Object -Property { $_.DistinguishedName.Length } -Descending |
    Remove-OpenADObject -Session $session
```

### Change

`src/PSOpenAD/OpenAD.cs`: `OpenADPrincipal.SID` is `SecurityIdentifier?` and is
`$null` when the object has no `objectSid`.

**Breaking change** for anyone dereferencing `.SID` without a null check; noted in
`CHANGELOG.md`.

### Tests

`tests/units/PSOpenADTests/OpenADPrincipalTests.cs`: a principal without
`objectSid` has a null SID, one with it keeps the value.

`tests/Get-OpenADGroupMember.Tests.ps1` adds "Returns a member that has no
objectSid", which creates a contact, puts it in a group and asserts the member
comes back with a null SID, so the command is covered against a server and not
just a constructed object. It fails on an unpatched module with
`ArgumentException: sid`.

---

## Not submitted: fix/unsigned-attribute-values

Held back deliberately. The change makes `ConvertToRawAttributeValue` write a
`uint`, and a uint backed enum such as `GroupType`, as the signed value an LDAP
integer carries, so `GroupType.Global | GroupType.IsSecurity` goes out as
`-2147483646` rather than `2147483650`, and a value read from a group can be
written back.

It has unit tests and is correct in principle, but there is no reproduction: the
AD used for validation accepts `2147483650` for `groupType` on both create and
modify, so there is no failing behaviour to point a PR at.
