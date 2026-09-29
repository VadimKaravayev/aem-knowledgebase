# AEM 6.5 operations over curl: the endpoints behind the consoles

Adobe's curl page for AEM 6.5, regrouped by the **servlet family** each command
hits, because that is what tells you which other operations are possible and
which arguments they take. Commands are Adobe's verbatim with `localhost:4502`
and `<user>:<password>` placeholders; documentation-derived, not re-run here.
Related: [[dispatcher-caching-and-invalidation]] (the invalidate endpoint in
context), [[shared-datastore-binaryless-replication]] (what replication agents
actually ship).

Two facts that govern everything below:

- **curl is a user.** Every call is authenticated and ACL-checked exactly as a
  browser session would be; there is no "admin API" bypass. Use a dedicated
  service user, not `admin`, for anything scripted.
- **Almost everything is the Sling POST servlet.** Form fields (`-F`) map to
  JCR properties or `:operation` verbs, so the same grammar that creates a
  folder creates a replication agent. Only Package Manager, the Web Console
  and a handful of `/bin/*` command servlets have their own vocabulary.

---

## Package Manager: `/crx/packmgr/service`

Three flavours of the same service, chosen by what you want back:

| Endpoint | Returns | Use |
|---|---|---|
| `/crx/packmgr/service.jsp?cmd=ls` | XML | listing (the only `.jsp` call) |
| `/crx/packmgr/service/.json/<pkgpath>?cmd=…` | JSON | scripting |
| `/crx/packmgr/service/console.html/<pkgpath>?cmd=…` | HTML log | human-readable output, e.g. `cmd=contents` |

`<pkgpath>` is the package's repository path, `/etc/packages/<group>/<name>.zip`.
Commands: `create`, `preview`, `contents`, `build`, `rewrap`, `upload`,
`install`, `uninstall`, `delete`, `replicate`.

```
# list
curl -u <user>:<password> http://<host>:<port>/crx/packmgr/service.jsp?cmd=ls
# create (definition only, empty filter until you edit it)
curl -u <user>:<password> -X POST http://localhost:4502/crx/packmgr/service/.json/etc/packages/mycontent.zip?cmd=create -d packageName=<name> -d groupName=<name>
# build / rewrap / preview / replicate: same shape, different cmd
curl -u <user>:<password> -X POST http://localhost:4502/crx/packmgr/service/.json/etc/packages/mycontent.zip?cmd=build
# upload (force=true overwrites an existing package of the same name)
curl -u <user>:<password> -F cmd=upload -F force=true -F package=@test.zip http://localhost:4502/crx/packmgr/service/.json
# install / uninstall / delete
curl -u <user>:<password> -F cmd=install   http://localhost:4502/crx/packmgr/service/.json/etc/packages/my_packages/test.zip
curl -u <user>:<password> -F cmd=uninstall http://localhost:4502/crx/packmgr/service/.json/etc/packages/my_packages/test.zip
curl -u <user>:<password> -F cmd=delete    http://localhost:4502/crx/packmgr/service/.json/etc/packages/my_packages/test.zip
# download is a plain GET of the node
curl -u <user>:<password> http://localhost:4502/etc/packages/my_packages/test.zip
```

Renaming is **not** a packmgr command; it is a Sling POST on the definition
node: `-Fname=<New Name> …/etc/packages/<group>/<pkg>.zip/jcr:content/vlt:definition`.
Same trick works for any other definition property (filters, description).

`upload` and `install` are separate steps; the UI's "upload and install" is
two calls. The `.json` responses carry `success: true|false` plus a `msg`,
check it, curl exits 0 on an HTTP 200 whose body says the install failed.

## Replication and activation

**Activate a page** (what the Publish button does):

```
curl -u <user>:<password> -X POST -F path="/content/path/to/page" -F cmd="activate"   http://localhost:4502/bin/replicate.json
curl -u <user>:<password> -X POST -F path="/content/path/to/page" -F cmd="deactivate" http://localhost:4502/bin/replicate.json
```

**Tree activation** is a different servlet with its own flags:

```
curl -u <user>:<password> -F cmd=activate -F ignoredeactivated=true -F onlymodified=true -F path=/content/geometrixx http://localhost:4502/etc/replication/treeactivation.html
```

`onlymodified=true` skips pages already published since their last change;
`ignoredeactivated=true` leaves deliberately unpublished pages alone. Without
both you re-activate the whole tree, which on a big site fills every agent
queue and the Dispatcher `.stat` storm that follows takes the site down.

**Packages** replicate through packmgr, not `replicate.json`:
`…/crx/packmgr/service/.json/<pkgpath>?cmd=replicate`.

**Agents** are ordinary `cq:Page` nodes under `/etc/replication/agents.author`
(or `agents.publish` for flush and reverse agents), so they are created with
the Sling POST servlet and inspected through the `.queue.json` selector on
their `jcr:content`:

```
# status + queue
curl -u <user>:<password> "http://localhost:4502/etc/replication/agents.author/publish/jcr:content.queue.json?agent=publish"
# pause / clear the queue
curl -u <user>:<password> -F "cmd=pause" -F "name=publish" http://localhost:4502/etc/replication/agents.author/publish/jcr:content.queue.json
curl -u <user>:<password> -F "cmd=clear" -F "name=publish" http://localhost:4502/etc/replication/agents.author/publish/jcr:content.queue.json
# create
curl -u <user>:<password> -F "jcr:primaryType=cq:Page" -F "jcr:content/jcr:title=new-replication" -F "jcr:content/sling:resourceType=/libs/cq/replication/components/agent" -F "jcr:content/template=/libs/cq/replication/templates/agent" -F "jcr:content/transportUri=http://localhost:4503/bin/receive?sling:authRequestLogin=1" -F "jcr:content/transportUser=admin" -F "jcr:content/transportPassword={DES}8aadb625ced91ac483390ebc10640cdf" http://localhost:4502/etc/replication/agents.author/replication99
# delete
curl -X DELETE http://localhost:4502/etc/replication/agents.author/replication99 -u <user>:<password>
```

The `{DES}…` transport password is the **encrypted** form; AEM encrypts a
plain value on save through the UI, but a POSTed plain string is stored as
is and the agent fails auth. Encrypt with the Crypto Support console
(`/system/console/crypto`) on the *same* instance, since the key is per
instance (differs after cloning unless the HMAC/master keys were copied).
Note the created agent has **no `enabled=true`**, add
`-F "jcr:content/enabled=true"` or it sits idle.

## Dispatcher flush

```
curl -H "CQ-Action: Activate"   -H "CQ-Handle: /content/test-site/" -H "CQ-Path: /content/test-site/" -H "Content-Length: 0" -H "Content-Type: application/octet-stream" http://<dispatcher-host>/dispatcher/invalidate.cache
curl -H "CQ-Action: Deactivate" -H "CQ-Handle: /content/test-site/" -H "CQ-Path: /content/test-site/" -H "Content-Length: 0" -H "Content-Type: application/octet-stream" http://<dispatcher-host>/dispatcher/invalidate.cache
```

Adobe's page shows `localhost:4502` as the target, which is a copy-paste
trap: the endpoint is served by the **Dispatcher module on Apache**, not by
AEM, and only from an IP on `/cache/allowedClients`. No `-u` because the
Dispatcher does not authenticate the call. Mechanics and the other flush
methods are in [[dispatcher-caching-and-invalidation]].

## Page commands: `/bin/wcmcommand`

One servlet, many `cmd` values, the same one the Sites console calls:

```
curl -u <user>:<password> -X POST -F cmd="lockPage"   -F path="/content/path/to/page" -F "_charset_"="utf-8" http://localhost:4502/bin/wcmcommand
curl -u <user>:<password> -X POST -F cmd="unlockPage" -F path="/content/path/to/page" -F "_charset_"="utf-8" http://localhost:4502/bin/wcmcommand
curl -u <user>:<password> -F cmd=copyPage -F destParentPath=/path/to/destination/parent -F srcPath=/path/to/source/location http://localhost:4502/bin/wcmcommand
```

Other `cmd` values the console uses through the same servlet (`createPage`,
`deletePage`, `movePage`, `createVersion`, `rollout`) take the same
`path`/`srcPath`/`destParentPath` vocabulary; watch the browser network tab
for the exact field set.

**Rollout** is the one Adobe now routes through the async framework, and it
is the only example on the page using a bearer token (the AEMaaCS form):

```
curl -H "Authorization: Bearer <token>" "https://<instance-url>/bin/asynccommand" \
   -d type=page \
   -d operation=asyncRollout \
   -d cmd=rollout \
   -d path="/content/<your-path>"
```

`type=page` is a **shallow** rollout; `type=deep` includes subpages.

## Users and groups: Granite authorizables

Creation goes to one servlet; everything afterwards is a Sling POST on the
authorizable's own node with the `.rw.html` selector:

```
# create
curl -u <user>:<password> -FcreateUser= -FauthorizableId=hashim -Frep:password=hashim http://localhost:4502/libs/granite/security/post/authorizables
curl -u <user>:<password> -FcreateGroup=group1 -FauthorizableId=testGroup1 http://localhost:4502/libs/granite/security/post/authorizables
# create with profile + membership in one call
curl -u <user>:<password> -FcreateUser=testuser -FauthorizableId=testuser -Frep:password=abc123 -Fmembership=contributor -Fprofile/gender=male http://localhost:4502/libs/granite/security/post/authorizables
# modify: profile property, membership from the user side, members from the group side
curl -u <user>:<password> -Fprofile/age=25 http://localhost:4502/home/users/h/hashim.rw.html
curl -u <user>:<password> -Fmembership=contributor -Fmembership=testgroup http://localhost:4502/home/users/t/testuser.rw.html
curl -u <user>:<password> -FaddMembers=testuser1    http://localhost:4502/home/groups/t/testGroup.rw.html
curl -u <user>:<password> -FremoveMembers=testuser1 http://localhost:4502/home/groups/t/testGroup.rw.html
# delete
curl -u <user>:<password> -FdeleteAuthorizable= http://localhost:4502/home/users/t/testuser
curl -u <user>:<password> -FdeleteAuthorizable= http://localhost:4502/home/groups/t/testGroup
```

- `createUser=` and `deleteAuthorizable=` are **flag fields**: the value is
  empty, presence is what matters.
- `membership` is **replace**, not append: POSTing one `membership` value
  drops the user from every other group. `addMembers`/`removeMembers` on the
  group are the incremental forms.
- The `/home/users/h/hashim` path assumes the classic first-letter
  intermediate folder. Since 6.1 users are created under **random
  intermediate folders** (`/home/users/a1B2c3…`), so look the path up
  (`/bin/querybuilder.json?type=rep:User&nodename=<id>`) rather than
  deriving it.

## Sling POST servlet on arbitrary content

```
# create a folder: any property you POST becomes a JCR property
curl -u <user>:<password> -F jcr:primaryType=sling:Folder http://localhost:4502/etc/test
# delete / move / copy via :operation
curl -u <user>:<password> -F :operation=delete http://localhost:4502/etc/test/test.properties
curl -u <user>:<password> -F":operation=move" -F":applyTo=/sourceurl" -F":dest=/target/parenturl/" https://localhost:4502/content
curl -u <user>:<password> -F":operation=copy" -F":applyTo=/sourceurl" -F":dest=/target/parenturl/" https://localhost:4502/content
# upload a file as nt:file; "*" = use the uploaded filename as node name
curl -u <user>:<password> -F"*=@test.properties"                 http://localhost:4502/etc/test
curl -u <user>:<password> -F"test2.properties=@test.properties"  http://localhost:4502/etc/test
curl -u <user>:<password> -F "*=@test.properties;type=text/plain" http://localhost:4502/etc/test
```

- `:dest` with a **trailing slash** means "into this parent, keep the name";
  without it, the target is the new node's full path (rename+move).
- Add `-F ":replace=true"` to copy/move onto an existing node, otherwise 500.
- On AEMaaCS the same calls only work on **mutable** paths; `/apps` and
  `/libs` are read-only at runtime, see [[runmode-sources-and-precedence]].

## OSGi Web Console

```
curl -u <user>:<password> -Faction=start http://localhost:4502/system/console/bundles/<bundle-name>
curl -u <user>:<password> -Faction=stop  http://localhost:4502/system/console/bundles/<bundle-name>
```

`<bundle-name>` is the **symbolic name** (`com.day.cq.wcm.cq-wcm-core`) or
the numeric bundle id. Other `action` values: `update`, `refresh`,
`uninstall`, `install` (with `-F bundlefile=@x.jar -F bundlestart=true`).
The console is admin-only by default and absent on AEMaaCS (Developer
Console replaces the read side; there is no start/stop at all).

## Review checklist

- Script returned 0 but nothing happened: read the JSON `success` field, or
  for Sling POST look for a 201/200 HTML status page with `Status: 500`.
- Replication agent created via curl never sends: `enabled` missing, or
  transport password POSTed in clear.
- User vanished from all groups after a "membership" call: `membership`
  replaces, use `addMembers` on the group.
- Flush curl returns 404 or 200 with no effect: it was sent to AEM instead of
  Apache, or the client IP is not in `/allowedClients`.
- Tree activation flooded the queues: missing `onlymodified` and
  `ignoredeactivated`.

## References
- [Using cURL with AEM (Adobe docs, AEM 6.5)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/sites/administering/operations/curl)
