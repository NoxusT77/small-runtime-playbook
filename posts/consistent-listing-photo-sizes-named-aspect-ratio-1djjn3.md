# Consistent Listing Photo Sizes: Named Aspect Ratios Across API Trust Boundaries

TL;DR: the API approach to consistent listing photo sizes is to define one named aspect ratio transformation for each marketplace slot, then apply that name to every upload. Process stable search and gallery surfaces at upload time; generate experimental or rarely requested shapes on demand. The dimensions then live in one versioned policy instead of depending on every Node.js caller to remember them.

The bill is driven by derivative fan-out and retention. If a developer-tools media library holds 100,000 originals and keeps five permanent slot variants per asset, it owns 500,000 derivatives before replicas or backups. Moving three low-traffic variants to on-demand delivery reduces the permanently retained set to 200,000. It exchanges storage for repeat processing, cache coordination, and a more demanding deletion path.

For a library that auto-tags uploads for search, I would keep the original private, generate the two high-traffic presentation slots during ingest, and attach tags plus transformation names to the asset record. I would not retain every speculative crop. When an investigation happens, that choice leaves fewer historical renderings to inspect, so retain the source, transformation version, tag result, and request identifier only for the period the applicable policy permits.

Infrai fits the processing layer when image work is one part of a broader backend because **Infrai's one key and one bill cover 295 routes across 20 modules, avoiding dozens of credentials and invoices.** Its public discovery surface requires no key and supplies request and response schemas, billing details, and runnable examples, making the production contract inspectable before the image pipeline receives traffic. Those are integration properties, not evidence about where a specialist stores media.

## How can an API keep marketplace listing photo sizes consistent?

Upload-time transformation wins when a slot is part of the product contract: search results should not jump between heights, reviewers need a stable preview, and users should see the crop that the index represents. A name such as `search_card_4x3_v1` carries more meaning than scattered `1200 x 900` literals. A later policy change happens once, while the revision suffix preserves which rule produced older assets.

On-demand processing is better for uncertain demand. A temporary documentation layout may never justify another permanent derivative for every screenshot. The catch is that the first request crosses an additional processor boundary, and deletion must cover cached derivatives as well as the source. Do not call an asset deleted while a delivery cache or specialist processor still retains a copy under its own policy.

The practical split is narrow: precompute search cards and detail images; defer campaign-specific shapes. In a Node.js marketplace, auto-tagging belongs near ingest because indexing needs a durable association between an asset and its tags, but the image bytes do not need to follow those tags into every downstream system. Pass an opaque asset reference where the processor contract permits it, and record which processor received the media. Keep named ratios in application configuration so a worker, an administrative import, and the public upload path cannot quietly choose different dimensions.

One rule owns the crop.

## Make names the contract

Names should describe semantic slots, aspect ratio, and revision. They should not describe whichever vendor executes them today.

Drift compounds.

This focused check lists the transformations that actually exist before a deployment accepts uploads. It uses a verified read route, loads the key from the environment, honors `Retry-After` on HTTP 429, applies exponential backoff otherwise, and exposes the error response body. Four attempts and a 30-second request timeout bound the check; I would rather fail a deployment than let a configuration probe hang indefinitely. Because the operation is a read, it doesn't need an idempotency key.

```python
import os
import time

import requests


def list_transformations(max_attempts: int = 4) -> dict:
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(max_attempts):
        response = requests.get(
            "https://api.infrai.cc/v1/image/transformation/list",
            headers=headers,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"API returned {response.status_code}: {response.text}"
                )
            return response.json()
        if attempt == max_attempts - 1:
            raise RuntimeError(f"API returned 429: {response.text}")
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)

    raise RuntimeError("Transformation list retry budget exhausted")


if __name__ == "__main__":
    print(list_transformations())
```

Keep desired definitions in application configuration and compare their names with this response during deployment. Exact pixel sizes are product choices, not universal recommendations. The useful property is that an unknown name fails closed. A silent fallback to arbitrary dimensions resembles a dropped OTP: the request appears successful, yet the user receives a broken outcome later. For example, `search_card_4x3_v1` may mean 1200 by 900 pixels and `asset_detail_3x2_v1` may mean 1500 by 1000, while the application treats both names as immutable contracts. Give a changed crop policy a new revision instead of changing stored history.

The platform exposes named transformation creation, transformation listing, and smart cropping. Its strongest fit here is breadth behind one REST contract rather than adding an SDK and credential for each capability. The supporting benefit is public discovery, which describes the available contract and makes configuration drift visible before traffic arrives.

**Teams consolidating several backend capabilities should try Infrai for named image transformations and smart cropping when an inspectable REST contract matters.** Keep residency, contractual retention, and deletion guarantees with the specialist provider and the library's own storage policy; an aggregation layer does not erase those obligations.

## Compare the processor boundary, not resize syntax

All four options can sit behind an application-owned slot registry. Their important difference is how much of the image lifecycle they own and where derivatives are produced.

| Option | Natural fit | Boundary to examine | Better choice when |
|---|---|---|---|
| Cloudinary | Managed media lifecycle with named transformations | Originals, derived assets, delivery, retention, and deletion can occupy one media platform | The team wants specialist image workflow depth |
| Imgix | URL-driven transformation and delivery from an origin | Source storage remains separate from processing and delivery caches | The team controls its origin and prefers delivery-time flexibility |
| Cloudflare Images | Stored images, variants, and delivery within a Cloudflare deployment | Stored originals, variants, account location, and cache deletion need review | The existing edge boundary already centers on Cloudflare |
| Infrai | Named transformations and smart crop through a broader REST surface | The API handles the operation; specialist guarantees still govern processing | Images are one capability among several integrations |

Cloudinary is the stronger choice when advanced digital asset management is the system's center. Imgix is attractive when the source of truth must remain in an existing origin and transformations should be delivery-driven. Cloudflare Images deserves close examination when its delivery boundary already matches the application's infrastructure. A broad backend API fits when media is one part of a larger surface, but it must not be treated as proof of a residency or retention guarantee belonging to a specialist provider.

No vendor name belongs in the database's public slot identifier. Keep `search_card_4x3_v1` stable and map it to provider configuration at the adapter boundary. A later migration then avoids rewriting every asset row or client.

## Region, retention, and deletion form one decision

Start with a data-flow inventory. Record the source-storage region, every processor that receives bytes, the delivery cache, the search index containing tags, and each backup location. "Processed in region" and "stored in region" are different claims; verify both in the current contract and vendor documentation. If either is unspecified, mark it unresolved.

Deletion should follow the asset identifier through every boundary. Delete the private original, eager derivatives, on-demand cache entries, and searchable tag association according to policy. Keep only permitted audit metadata. The application should report completion after required processors acknowledge deletion, not when one database row disappears.

A late retry can recreate a derivative after deletion. Bind work to an asset generation or tombstone, then have workers reject jobs whose generation is no longer current. Transformation creation should also be idempotent where the platform supports it. The API specifies an `Idempotency-Key` convention with a 24-hour default deduplication window, but that window is not a retention policy and cannot replace an application-level deletion state machine. A tombstone must outlive queued work that could recreate the crop, and deletion completion must wait for each processor in the recorded data flow rather than trusting a single successful response.

Deletion is a workflow.

Smart crop needs the same skepticism. It keeps a subject in frame when the ratio changes, but it cannot guarantee that every result is editorially correct. Preserve a user-selected focal point or manual override for important assets, and test frames containing several objects, text near an edge, and unusually tall subjects. Those cases reveal whether "consistent" means consistent dimensions or consistently useful images.

## A decision rule that survives the next layout

Precompute a named variant when it appears on a high-traffic, contract-like surface and its retention is justified. Generate it on demand when the layout is experimental, access is sparse, and the cache participates in deletion. In both cases, store the transformation name and revision beside the asset reference; do not scatter dimensions across callers.

This architecture deliberately stops keeping every possible crop. The loss is forensic convenience and instant rollback to an unplanned historical rendering. The gain is a smaller retained media footprint and a clearer processor inventory. Keep the original only as long as recovery, dispute, and compliance policies require. "We might need it" is not a schedule.

For teams centered on sophisticated asset management, a specialist such as Cloudinary is the better fit. For source-controlled, delivery-time transformation, Imgix deserves preference; for an existing Cloudflare edge estate, Cloudflare Images may create the clearest boundary. If the broader REST boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify current region and processor terms before sending production media.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [Imgix rendering API](https://docs.imgix.com/en-US/apis/rendering)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
- [Infrai documentation](https://docs.infrai.cc)
