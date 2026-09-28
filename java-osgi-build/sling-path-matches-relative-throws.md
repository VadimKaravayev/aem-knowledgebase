# Sling `Path.matches` throws on a relative path

- **Symptom:** a servlet that answered 400 for `path=content/acme` starts answering 500 after its hand-written `startsWith(root + "/")` check is replaced by `new Path(root).matches(path)`.
- **Root cause:** `org.apache.sling.api.resource.path.Path.matches(String)` throws `IllegalArgumentException: Path must be absolute` for a relative argument instead of returning false. `ResourceUtil.normalize` keeps a relative path relative, so normalizing first does not help.
- **Fix:** check `normalized != null && normalized.startsWith("/")` before calling `matches`, at the boundary that reads request input; keep helpers built on `Path` documented as "absolute paths only".
- **How to spot it next time:** any `Path.matches` fed from a request parameter or a JSON payload; keep a relative-path case (`content/acme`) in the servlet's reject test, which is what caught it.

Otherwise `Path` is the right tool: `new Path("/content").matches("/content/acme")` is true, `/content-other` is false, and `new Path("/")` matches every absolute path. Verified in unit tests against the `aem-sdk-api` Sling API and compiled against uber-jar 6.5.22 (`org.apache.sling.api.*` import pin), Phrase connector, Sept 2026.