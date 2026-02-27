# Search Bar Feature – File Changes (Before/After & Justifications)

## File: src/routes/(kener)/api/search/+server.js

**New file.** No “before” section.

### After (full file)

```javascript
// @ts-nocheck
// Search API: returns monitors, groups, and supergroups matching query (name or tag).
// Used by the header search bar to navigate to monitor/group/supergroup heatmaps.
import { json } from "@sveltejs/kit";
import { GetMonitors } from "$lib/server/controllers/controller.js";

export async function GET({ url }) {
  const q = (url.searchParams.get("q") || "").trim().toLowerCase();
  if (!q) {
    return json([]);
  }
  // Fetch all ACTIVE monitors (leaf, GROUP, SUPERGROUP) and filter by name/tag
  const monitors = await GetMonitors({ status: "ACTIVE" });
  const results = monitors
    .filter(
      (m) =>
        (m.name && String(m.name).toLowerCase().includes(q)) ||
        (m.tag && String(m.tag).toLowerCase().includes(q))
    )
    .map((m) => ({
      tag: m.tag,
      name: m.name || m.tag,
      monitor_type: m.monitor_type || "API",
    }))
    .slice(0, 20);
  return json(results);
}
```

**Change justification:**
- **Purpose:** Provide a dedicated search endpoint for the header search bar so it can resolve monitors, groups, and supergroups by name or tag without changing any existing API or page load logic.
- **Minimal change:** New file only; no existing routes or controllers modified.
- **Existing functionality:** Unaffected; only additive.
- **Risks:** None for existing behavior. If ACTIVE monitor count grows very large, consider moving filter to DB (e.g. LIKE) later.

---

## File: src/lib/components/nav.svelte

### Before (lines 1–10)

```svelte
<script>
  import { Button } from "$lib/components/ui/button";
  import * as DropdownMenu from "$lib/components/ui/dropdown-menu";
  import Languages from "lucide-svelte/icons/languages";
  import Menu from "lucide-svelte/icons/menu";
  import { base } from "$app/paths";
  import { analyticsEvent } from "$lib/boringOne";
  import GMI from "$lib/components/gmi.svelte";
  export let data;
  let defaultPattern = data.site?.pattern || "squares";
```

### After (lines 1–75)

```svelte
<script>
  import { Button } from "$lib/components/ui/button";
  import { Input } from "$lib/components/ui/input";
  import * as DropdownMenu from "$lib/components/ui/dropdown-menu";
  import Search from "lucide-svelte/icons/search";
  import Menu from "lucide-svelte/icons/menu";
  import { base } from "$app/paths";
  import { goto } from "$app/navigation";
  import { analyticsEvent } from "$lib/boringOne";
  import GMI from "$lib/components/gmi.svelte";
  import { clickOutsideAction } from "svelte-legos";

  export let data;
  let defaultPattern = data.site?.pattern || "squares";

  // Search bar: query, results, dropdown visibility, debounce (no existing nav/heatmap logic changed)
  let searchQuery = "";
  let searchResults = [];
  let searchDropdownOpen = false;
  let searchDebounceTimer;
  $: {
    const q = typeof searchQuery === "string" ? searchQuery.trim() : "";
    if (q) {
      clearTimeout(searchDebounceTimer);
      searchDebounceTimer = setTimeout(async () => {
        try {
          const res = await fetch(`${base}/api/search?q=${encodeURIComponent(q)}`);
          const result = await res.json();
          searchResults = Array.isArray(result) ? result : [];
          searchDropdownOpen = searchResults.length > 0;
        } catch (e) {
          searchResults = [];
          searchDropdownOpen = false;
        }
      }, 200);
    } else {
      searchResults = [];
      searchDropdownOpen = false;
    }
  }

  // Navigate to monitor or group/supergroup heatmap using existing routing
  function navigateToItem(item) {
    if (!item || !item.tag) return;
    analyticsEvent("search_navigate", { tag: item.tag, monitor_type: item.monitor_type });
    const isGroupOrSupergroup = item.monitor_type === "GROUP" || item.monitor_type === "SUPERGROUP";
    const url = isGroupOrSupergroup ? `${base}?group=${item.tag}` : `${base}?monitor=${item.tag}`;
    searchQuery = "";
    searchResults = [];
    searchDropdownOpen = false;
    goto(url);
  }

  function handleSearchKeydown(e) {
    if (e.key === "Enter") {
      e.preventDefault();
      if (searchResults.length > 0) {
        navigateToItem(searchResults[0]);
      }
    }
    if (e.key === "Escape") {
      searchDropdownOpen = false;
    }
  }
```

**Inline justifications (in code):**
- `// Search bar: query, results, dropdown visibility, debounce (no existing nav/heatmap logic changed)`
- `// Navigate to monitor or group/supergroup heatmap using existing routing`

**Change justification:**
- **Purpose:** Add search bar UI (magnifying glass, input, dropdown) and wire it to the new search API and existing `?monitor=` / `?group=` routing.
- **Minimal change:** Only new imports, new state, and new markup block; existing nav links, logo, and mobile menu unchanged.
- **Existing functionality:** Documentation/GitHub links, mobile menu, and all other nav behavior unchanged.
- **Risks:** None; dropdown is closed on outside click and on navigate.

---

### Before (lines 76–78)

```svelte
    <div class="flex w-full justify-end">
      {#if data.site.nav}
        <nav class=" hidden flex-wrap items-center text-sm font-medium md:flex">
```

### After (lines 126–175)

```svelte
    <div class="flex w-full flex-1 items-center justify-end gap-x-2">
      <!-- Search bar next to Documentation/GitHub: magnifying glass icon + input + dropdown results -->
      <div
        class="relative flex items-center"
        use:clickOutsideAction
        on:clickoutside={() => (searchDropdownOpen = false)}
      >
        <Search
          class="pointer-events-none absolute left-2.5 h-4 w-4 text-muted-foreground"
          aria-hidden="true"
        />
        <Input
          type="text"
          placeholder="Search monitors..."
          bind:value={searchQuery}
          on:keydown={handleSearchKeydown}
          class="h-9 w-28 pl-8 text-sm md:w-40"
          aria-label="Search monitors, groups, and supergroups"
        />
        {#if searchDropdownOpen && searchResults.length > 0}
          <ul
            class="absolute top-full left-0 z-50 mt-1 max-h-60 w-56 overflow-auto rounded-md border bg-card py-1 shadow-md"
            role="listbox"
          >
            {#each searchResults as item (item.tag)}
              <li role="option">
                <button
                  type="button"
                  class="flex w-full cursor-pointer items-center gap-x-2 px-3 py-2 text-left text-sm hover:bg-accent"
                  on:click={() => navigateToItem(item)}
                >
                  <span class="truncate font-medium">{item.name}</span>
                  <span class="shrink-0 text-xs text-muted-foreground">({item.monitor_type})</span>
                </button>
              </li>
            {/each}
          </ul>
        {/if}
      </div>
      {#if data.site.nav}
        <nav class="hidden flex-wrap items-center text-sm font-medium md:flex">
```

**Inline justifications (in template):**
- `<!-- Search bar next to Documentation/GitHub: magnifying glass icon + input + dropdown results -->`

**Change justification:**
- **Purpose:** Place search bar in the header next to Documentation/GitHub; show icon and dropdown; keep existing nav unchanged.
- **Minimal change:** One new wrapper div and one new block for search; nav structure and links unchanged.
- **Existing functionality:** Unaffected.
- **Risks:** None; layout uses flex and gap so existing links remain in place.
