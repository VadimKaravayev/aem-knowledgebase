# IntelliJ IDEA + Jupyter: "server is not started" means the module isn't a Python module

Running `.ipynb` notebooks inside IntelliJ IDEA Ultimate (not PyCharm) with a uv-managed
`.venv` — see [uv-python-project-basics.md](uv-python-project-basics.md). The IDE's bundled
Jupyter support launches a *managed* Jupyter server from the project interpreter; when it
can't resolve that interpreter every cell fails with **"Jupyter server is not started"**, with
no further hint in the UI. Verified IntelliJ IDEA 2026.3, jupyterlab 4.6 / jupyter_server 2.21,
Python 3.13, macOS.

## Root cause

IntelliJ resolves the notebook interpreter **from the module type, not from the SDK you
attached**. A project created with File → New → Project while the *Java* generator was
selected (the IDE default) gets an `.iml` with `type="JAVA_MODULE"`. You can then point that
module's SDK at a Python interpreter and code completion works, but the Jupyter plugin's
`JupyterPyUtils.detectSdk` only looks at Python modules (or Python facets) and gives up:

```
WARN - #c.i.j.p.h.JupyterPyUtils - No suitable Python SDK found for module: <name>
INFO - JupyterLabManagedServerController - Jupyter server <name> state changed to JupyterServerStartError(...)
```

That line is in `~/Library/Logs/JetBrains/IntelliJIdea<ver>/idea.log` (Help → Show Log in
Finder). The notebook file, the venv and the Jupyter install are all fine — the same server
starts from a shell with `.venv/bin/jupyter server`.

## Fix: make it a `PYTHON_MODULE`

The IDE has no "change module type" action, so either replace the module or edit the file.

**Edit `.idea/<name>.iml`** (quickest; IDE picks it up on File → Reload All from Disk or reopen):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<module type="PYTHON_MODULE" version="4">
  <component name="NewModuleRootManager">
    <content url="file://$MODULE_DIR$">
      <sourceFolder url="file://$MODULE_DIR$/src" isTestSource="false" />
      <excludeFolder url="file://$MODULE_DIR$/.venv" />
    </content>
    <orderEntry type="jdk" jdkName="~/path/to/project/.venv" jdkType="Python SDK" />
    <orderEntry type="sourceFolder" forTests="false" />
  </component>
</module>
```

`jdkName` must match the SDK's `<name>` in
`~/Library/Application Support/JetBrains/IntelliJIdea<ver>/options/jdk.table.xml` exactly
(IntelliJ names venv SDKs by their `~`-relative path). Drop the `inherit-compiler-output` /
`<exclude-output />` lines the Java generator wrote.

**Via the UI** (File → Project Structure → Modules): remove the existing module with `−`
(this unregisters the `.iml`, files are untouched), `+` → New Module → **Python**, content
root = project folder, module SDK = the `.venv` interpreter; on the Sources tab mark `src`
as Sources and `.venv` as Excluded; Apply and reload.

**Avoid it next time:** File → New → Project → pick **Python** in the generator list, or
File → Open on a folder that already has `pyproject.toml` — recent IntelliJ detects the uv
project and creates a Python module itself.

## Gotchas

- `.idea/` is normally gitignored, so the fix is per-clone; every fresh checkout of a
  Java-generated project needs it once. Symptom recurs silently on a new machine.
- `misc.xml` may still say `languageLevel="JDK_23"` and `project-jdk-type="Python SDK"` on
  the same line — harmless, leave it.
- **Prove the notebook itself is fine before touching the IDE:**
  `uv run jupyter nbconvert --to notebook --execute --allow-errors --output-dir /tmp nb.ipynb`
  runs every cell headlessly on the project kernel and records per-cell errors in the output
  copy. Zero errors there + "server not started" in the IDE = IDE config, not code.
- `uv run jupyter kernelspec list` should show exactly one `python3` under `.venv/share/jupyter/kernels`;
  the kernel's `language_info.version` baked into the `.ipynb` by whoever last saved it
  (e.g. 3.12) doesn't matter — the kernelspec wins.
- The `notebook` package is **not** installed by `jupyterlab`; IntelliJ's managed server
  only needs `jupyter_server`, which jupyterlab pulls in, so don't add `notebook` to fix this.
