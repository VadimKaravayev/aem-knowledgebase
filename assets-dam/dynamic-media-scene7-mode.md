# Dynamic Media – Scene7 Mode (AEM 6.5)

How AEM 6.5 offloads asset rendition generation and delivery to Adobe's cloud-hosted Dynamic Media (Scene7) service, and how to wire it up. Sources: [AEM 6.5 config-dms7 docs](https://github.com/AdobeDocs/experience-manager-65.en/blob/main/help/assets/config-dms7.md), [troubleshoot-dms7 docs](https://github.com/AdobeDocs/experience-manager-65.en/blob/main/help/assets/troubleshoot-dms7.md).

---

## What it is / the problem it solves

Two Dynamic Media modes exist on 6.5: **Hybrid** (`dynamicmedia` run mode — renditions generated locally on the AEM instance) and **Scene7/cloud mode** (`dynamicmedia_scene7` run mode — renditions generated and served entirely by Adobe's cloud).

In Scene7 mode:
- AEM stores only the **master/original asset**.
- On upload/activation, the asset is **replicated to the Scene7 cloud tenant**.
- Scene7 generates every rendition on demand (resize, crop, ICC color-managed output, video encoding — 50+ variants is trivial) and serves them via its **own built-in Akamai CDN**.
- AEM's local storage/CPU never touches rendition generation — this is what makes it the right answer at tens-of-TB scale (avoids a 50x local storage multiplier) and replaces bespoke rendition tooling + a separate CDN/S3 delivery setup in one move.

**One-line mental model:** *AEM = source-of-truth authoring + master asset; Scene7 = the rendition factory + CDN, living entirely outside the AEM JVM.*

## Configuration steps

1. **Start Author with the Scene7 run mode** — Author-only, never Publish:
   ```
   java -jar cq-quickstart-6.5.0.jar -gui -r author,dynamicmedia_scene7 -p 4502
   ```
2. **Create the Cloud Services config**: Tools → Cloud Services → Dynamic Media Configuration → select **global** in the left pane → Create. Fill in Title, the Dynamic Media account email/password/region (from Adobe provisioning), then **Connect to Dynamic Media**.
3. **Set the account password** (8–25 chars; needs upper/lowercase, a number, one of `# $ & . - _ : { }`) → Done → Save.
4. **Configure publishing settings**: Company (account name), **Company Root Folder Path** (where synced assets land in Scene7's folder structure), **Publishing Assets** mode (*Immediately* / *Upon Activation* / *Selective Publish*), **Sync all content** toggle.
5. **Save.**
6. **Verify**: account appears under Available Configurations in Cloud Services; the **Dynamic Media Asset Activation (scene7)** replication agent is enabled (this is what actually pushes assets to Scene7 on activation).

**Multi-environment gotcha:** this config is per-AEM-environment — dev/stage/prod each need their own Cloud Services configuration; it is not shared automatically.

## Exam framing — when Scene7 mode is the answer

Scene7/cloud mode is the correct choice when a scenario combines: **large/growing asset volume** (tens of TB+), **many renditions per asset** with color-accuracy requirements, and an **existing separate CDN/delivery setup** (e.g. S3) that should be consolidated. It beats the alternatives because:

| Alternative | Why it's usually wrong at this scale |
|---|---|
| **Hybrid mode** (`dynamicmedia` run mode) | Generates/stores renditions on the AEM instance itself — at 30TB with 50 renditions/asset this is an enormous local processing + storage burden. |
| **Asset Share Commons** | A reference search/sharing portal — doesn't generate renditions or manage color accuracy at all. |
| **Offloading server for renditions** | Spreads workflow processing across multiple AEM instances, but renditions still live in AEM infrastructure — doesn't solve the underlying scalability problem, just distributes it. |

See [brand-portal.md](brand-portal.md) for the related-but-distinct concern of distributing assets to **external** partners (Scene7/Dynamic Media is about internal rendition generation + delivery; Brand Portal is about external audience access).
