# NetBox Plugin Conversion Analysis for Rackula

**Date:** 2026-01-14
**Status:** Evaluation/Research

## Executive Summary

**Converting Rackula to a NetBox plugin is technically feasible but represents a significant undertaking.** The core challenge is that Rackula is a client-side SPA built with Svelte 5, while NetBox plugins follow a server-rendered Django architecture with HTMX for progressive enhancement.

---

## 1. Current Rackula Architecture

| Aspect | Technology |
|--------|-----------|
| **Framework** | Svelte 5 (runes: `$state`, `$derived`, `$effect`) |
| **Language** | TypeScript (strict mode) |
| **Build** | Vite 7.2 |
| **Rendering** | Pure SVG with panzoom library |
| **State** | Client-side only (Svelte 5 runes) |
| **Persistence** | File download (.Rackula.zip), URL sharing |
| **Backend** | None (100% client-side) |

**Key files:**
- `src/lib/stores/layout.svelte.ts` — 64KB central state management
- `src/lib/types/index.ts` — Complete type system
- `src/lib/schemas/index.ts` — Zod validation schemas
- `src/lib/utils/export.ts` — 50KB export engine

---

## 2. NetBox Plugin Requirements

Based on [NetBox Plugin Development docs](https://netboxlabs.com/docs/netbox/plugins/development/):

| Requirement | NetBox Plugin |
|-------------|---------------|
| **Language** | Python 3.12+ |
| **Framework** | Django (server-rendered) |
| **Frontend** | Django templates + HTMX + vanilla JS |
| **State** | PostgreSQL database via Django ORM |
| **Models** | Must extend NetBox's model framework |
| **URLs** | Restricted to `/plugins/` prefix |

NetBox now uses [HTMX globally](https://github.com/netbox-community/netbox/issues/14736) (merged in v4.0) for partial page updates, providing a middle ground between traditional server rendering and SPAs.

---

## 3. Conversion Complexity Assessment

### What Would Need Complete Rewriting (~80% of codebase)

| Component | Rackula | NetBox Plugin |
|-----------|---------|---------------|
| **State management** | Svelte 5 runes | Django models + PostgreSQL |
| **UI components** | 85 Svelte components | Django templates + HTMX |
| **Canvas/drag-drop** | panzoom + SVG | Fabric.js or Konva.js |
| **Export engine** | Client-side jsPDF | Server-side PDF generation |
| **Validation** | Zod schemas | Django serializers |
| **Build system** | Vite | Django asset pipeline |

### What Could Be Adapted (~20%)

- **Type definitions** → Django model design
- **Collision detection algorithm** → Pure logic, language-agnostic
- **NetBox import utilities** → Already NetBox-compatible
- **Device images** → Asset reuse
- **Data models** → Already use NetBox-compatible field names

---

## 4. NetBox Plugin Precedents

There are relevant existing plugins that demonstrate what's achievable:

1. **[netbox-floorplan-plugin](https://github.com/netbox-community/netbox-floorplan-plugin)** — Graphical floorplan editor with drag-and-drop, SVG export (~43% JavaScript)

2. **[netbox-topology-views](https://github.com/netbox-community/netbox-topology-views)** — Topology visualization with export to draw.io/PNG

3. **[Drag-and-drop Rack Elevations](https://plugin-ideas.netbox.dev/ideas/PLUGINS-I-15)** — $1,500 bounty plugin proposal for exactly this functionality (not yet implemented)

The floorplan plugin proves that rich interactive visualization is achievable within NetBox's architecture, using JavaScript canvas libraries alongside Django.

---

## 5. Conversion Options

### Option A: Full Plugin Conversion (High Effort)

**Approach:** Complete rewrite as Django plugin with JavaScript canvas

**Technology stack:**
- Python/Django for models, views, API
- [Fabric.js](https://fabricjs.com/) or [Konva.js](https://konvajs.org/) for canvas (both framework-agnostic)
- HTMX for Django template integration
- Django REST Framework for API endpoints

**Effort estimate:** 12-16 weeks
**Pros:** Full NetBox integration, single source of truth
**Cons:** Loses portability, requires NetBox instance, significant rewrite

### Option B: Embedded iframe/Micro-frontend (Medium Effort)

**Approach:** Keep Svelte app, embed in NetBox via plugin

**Implementation:**
- NetBox plugin provides an iframe page
- Rackula runs as standalone Svelte app inside iframe
- API bridge syncs data between Rackula and NetBox

**Effort estimate:** 4-6 weeks
**Pros:** Minimal Rackula changes, keeps all features
**Cons:** Not "native" feel, authentication complexity

### Option C: API Integration (Low Effort)

**Approach:** Keep Rackula standalone, add NetBox API sync

**Implementation:**
- Add NetBox API client to Rackula
- Import devices/racks directly from NetBox instance
- Export layouts back as custom fields or attachments

**Effort estimate:** 2-3 weeks
**Pros:** No architecture changes, works today
**Cons:** Two separate applications to maintain

---

## 6. Recommended Approach

Given Rackula's architecture and NetBox's plugin system, we recommend a **phased approach**:

### Phase 1: API Integration (Option C)
- Add NetBox API import/export to existing Rackula
- Users can pull live data from their NetBox instance
- Minimal disruption, immediate value

### Phase 2: Evaluate Full Plugin Need
- Monitor the [drag-and-drop rack elevation bounty](https://netboxlabs.com/blog/announcing-the-netbox-plugin-bounty-program/) progress
- If NetBox community builds this, Rackula may not need conversion
- If demand exists, proceed to Phase 3

### Phase 3: Canvas-based Plugin (if needed)
- Port rendering logic to Fabric.js/Konva.js
- Build Django models from Rackula's type system
- Use HTMX for UI integration
- Keep Rackula as standalone option for offline/portability use cases

---

## 7. Technical Considerations for Full Conversion

If proceeding with full conversion, here are key decisions:

### Canvas Library Choice

| Library | Pros | Cons |
|---------|------|------|
| **Fabric.js** | More features, better SVG support | Heavier, steeper learning curve |
| **Konva.js** | Better performance, simpler API | Less mature SVG handling |

Both are framework-agnostic and work with HTMX-based Django templates.

### Django Model Design

Rackula's type system already uses NetBox-compatible field names (`slug`, `u_height`, `is_full_depth`), making model translation straightforward:

```python
# Example Django model (derived from Rackula types)
class PlacedDevice(models.Model):
    rack = models.ForeignKey('dcim.Rack', on_delete=models.CASCADE)
    device_type = models.ForeignKey('dcim.DeviceType', on_delete=models.CASCADE)
    position = models.PositiveIntegerField()
    face = models.CharField(choices=[('front', 'Front'), ('rear', 'Rear'), ('both', 'Both')])
    name = models.CharField(max_length=100, blank=True)
    # etc.
```

---

## 8. Summary

| Option | Effort | Integration Level | Recommendation |
|--------|--------|-------------------|----------------|
| **A: Full Plugin** | High (12-16 weeks) | Native | Only if NetBox-native is required |
| **B: iframe Embed** | Medium (4-6 weeks) | Hybrid | Good compromise |
| **C: API Sync** | Low (2-3 weeks) | Standalone | **Start here** |

The most pragmatic path is to first add API integration (Option C), then evaluate whether full conversion is warranted based on user demand and the progress of NetBox's own [drag-and-drop rack elevation initiative](https://plugin-ideas.netbox.dev/ideas/PLUGINS-I-15).

---

## References

- [NetBox Plugin Development Documentation](https://netboxlabs.com/docs/netbox/plugins/development/)
- [NetBox HTMX Integration (Issue #14736)](https://github.com/netbox-community/netbox/issues/14736)
- [NetBox Floorplan Plugin](https://github.com/netbox-community/netbox-floorplan-plugin)
- [Drag-and-drop Rack Elevations Plugin Bounty](https://plugin-ideas.netbox.dev/ideas/PLUGINS-I-15)
- [NetBox Plugin Bounty Program Announcement](https://netboxlabs.com/blog/announcing-the-netbox-plugin-bounty-program/)
- [Fabric.js](https://fabricjs.com/)
- [Konva.js](https://konvajs.org/)
