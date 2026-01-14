# NetBox Plugin Conversion Analysis

**Project:** Rackula - Rack Layout Designer  
**Analysis Date:** 2026-01-14  
**Repository:** https://github.com/jirivrany/Rackula

---

## Executive Summary

**Question:** Can Rackula be converted to a NetBox plugin, or can a similar NetBox plugin be created using NetBox models as data sources?

**Answer:** **YES - Both approaches are feasible**, with different trade-offs:

1. **Option A: NetBox Plugin with Rackula Integration** (Recommended)
   - Create a NetBox plugin that embeds Rackula as a visualization tool
   - Use NetBox's existing rack/device data as the source of truth
   - Leverage Rackula's superior visualization capabilities
   - **Effort:** Medium (4-6 weeks for MVP)

2. **Option B: Standalone NetBox Plugin** (Alternative)
   - Build a new rack visualization plugin from scratch using NetBox models
   - Native Django/NetBox integration
   - **Effort:** High (8-12 weeks for feature parity)

**Recommendation:** Option A (NetBox Plugin with Rackula Integration) provides the fastest path to value while leveraging both systems' strengths.

---

## 1. Current State Analysis

### 1.1 Rackula Architecture

**Technology Stack:**
- **Frontend:** Svelte 5 (SPA with runes), TypeScript strict mode
- **Rendering:** SVG-based rack visualization
- **Data Model:** NetBox-compatible schema (intentionally designed for future integration)
- **Storage:** Client-side only (browser localStorage, file download/upload)
- **Deployment:** Static site (GitHub Pages for dev, Docker for prod)
- **No Backend:** Completely client-side application

**Key Features:**
- Drag-and-drop device placement
- Front/rear rack views with dual-view mode
- Device images with bundled library (from NetBox devicetype-library)
- Collision detection for full-depth and half-depth devices
- Export to PNG, PDF, SVG, CSV
- Undo/redo with command pattern
- Airflow visualization
- Multi-rack support (v0.6.0)
- Port-to-port connection tracking (MVP)

**Data Model Compatibility:**
Rackula was **explicitly designed with NetBox compatibility**:

```typescript
// Already NetBox-compatible field names (snake_case)
interface DeviceType {
  slug: string;
  manufacturer?: string;
  model?: string;
  part_number?: string;
  u_height: number;
  is_full_depth?: boolean;
  weight?: number;
  weight_unit?: 'kg' | 'lb';
  airflow?: Airflow;  // NetBox-compatible enum
  // ... etc
}
```

Key indicators of NetBox alignment:
- Field naming convention (snake_case matches NetBox ORM)
- Airflow enum matches NetBox's airflow choices
- Interface templates follow NetBox's device type structure
- Cable/connection model parallels NetBox's cable tracking
- Form factor enums match NetBox rack types

### 1.2 Existing NetBox Integration

**Current Integration Points:**

1. **Device Type Import Scripts:**
   - `scripts/import-netbox-devices.ts` - Imports single devices from NetBox devicetype-library
   - `scripts/bulk-import-netbox.ts` - Bulk imports 500+ devices from multiple vendors
   - Downloads device YAML definitions and elevation images
   - Generates TypeScript device library code

2. **Device Type Library Source:**
   - Uses NetBox Community Device Type Library as upstream source
   - https://github.com/netbox-community/devicetype-library
   - Bundled images processed from NetBox elevation images
   - Covers 15+ vendors (Dell, HPE, Ubiquiti, Mikrotik, Cisco, etc.)

3. **Schema Compatibility:**
   - Data model intentionally mirrors NetBox's device/rack structure
   - Field names use NetBox conventions (snake_case)
   - Airflow, weight units, form factors match NetBox enums

**Current Limitations:**
- No live NetBox API connection
- Device data is imported statically at build time
- No synchronization with NetBox database
- Cannot read existing NetBox rack layouts

---

## 2. NetBox Plugin Architecture Overview

### 2.1 What is a NetBox Plugin?

NetBox plugins are Django applications that extend NetBox's functionality. They integrate into NetBox's UI, use its data models, and follow its authentication/permission system.

**Plugin Requirements:**
- Python Django application
- Uses NetBox's ORM models (Rack, Device, DeviceType, etc.)
- Integrates into NetBox's navigation and views
- Follows NetBox's permission model
- Can add custom views, API endpoints, templates

**Key NetBox Models (Relevant to Rackula):**
```python
# NetBox DCIM models
from dcim.models import (
    Rack,           # Physical rack
    Device,         # Installed device
    DeviceType,     # Device template/definition
    Manufacturer,   # Vendor
    Site,           # Data center location
    Location,       # Room/area within site
    RackRole,       # Rack categorization
    DeviceRole,     # Device categorization
    Interface,      # Network ports
    Cable,          # Physical connections
)
```

### 2.2 NetBox Plugin Development Stack

**Required Technologies:**
- Python 3.10+
- Django 4.2+ (NetBox's web framework)
- NetBox 4.0+ (current: 4.2.x LTS)
- PostgreSQL (NetBox's database)

**Plugin Structure:**
```
netbox-plugin-rackula/
├── netbox_rackula/
│   ├── __init__.py
│   ├── navigation.py       # Add menu items
│   ├── views.py            # Django views
│   ├── urls.py             # URL routing
│   ├── api/                # REST API endpoints
│   ├── templates/          # Django templates
│   ├── static/             # CSS, JS, images
│   └── models.py           # Optional custom models
├── setup.py
├── README.md
└── LICENSE
```

---

## 3. Conversion Approaches

### 3.1 Option A: NetBox Plugin with Rackula Integration (RECOMMENDED)

**Concept:** Create a NetBox plugin that embeds the Rackula SPA as an iframe/webview, feeding it NetBox data via a translation layer.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                      NetBox Instance                         │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  NetBox Rackula Plugin (Django)                        │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │  Views (Python)                                   │  │  │
│  │  │  - Rack selection view                            │  │  │
│  │  │  - Data serialization endpoint                    │  │  │
│  │  │  - Import/export handlers                         │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │  API Layer (Python)                               │  │  │
│  │  │  - /api/plugins/rackula/rack/<id>/layout/         │  │  │
│  │  │  - NetBox models → Rackula Layout JSON            │  │  │
│  │  │  - DeviceType enrichment (images, colors)        │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
│                              ▼                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Embedded Rackula UI (Static Svelte build)            │  │
│  │  - Rack visualization                                  │  │
│  │  - Drag-and-drop (read-only or write-back mode)       │  │
│  │  - Export to image/PDF                                 │  │
│  │  - Communicates with plugin API via fetch()           │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           ▼
          ┌────────────────────────────────┐
          │   NetBox Database (PostgreSQL)  │
          │   - dcim_rack                   │
          │   - dcim_device                 │
          │   - dcim_devicetype             │
          └────────────────────────────────┘
```

**Implementation Steps:**

1. **Plugin Scaffold (Week 1)**
   - Create NetBox plugin boilerplate
   - Register navigation menu item ("Rack Visualizer")
   - Create basic Django view

2. **Data Translation Layer (Week 2)**
   - Build API endpoint: `GET /api/plugins/rackula/rack/<id>/layout/`
   - Serialize NetBox Rack + Devices → Rackula Layout JSON
   - Map NetBox DeviceType → Rackula DeviceType
   - Handle device positioning (NetBox uses position/face fields)

3. **Embed Rackula UI (Week 3)**
   - Build Rackula in "embeddable" mode (single HTML + bundled JS/CSS)
   - Serve static build from plugin's `static/` directory
   - Create Django template that loads Rackula
   - Pass rack ID and API endpoint URLs to Rackula via data attributes

4. **Bidirectional Sync (Week 4-5 - Optional for MVP)**
   - Add `POST /api/plugins/rackula/rack/<id>/layout/` endpoint
   - Validate and import Rackula changes back to NetBox
   - Handle device position updates, new device placements
   - Transaction safety and validation

5. **Polish & Testing (Week 6)**
   - Permission checks (NetBox's built-in RBAC)
   - Error handling
   - Documentation
   - Unit tests and integration tests

**Pros:**
- ✅ Reuses Rackula's proven visualization code (no reimplementation)
- ✅ Faster development (4-6 weeks for MVP)
- ✅ Rackula features available immediately (export, airflow, etc.)
- ✅ Minimal changes to Rackula codebase (build as embeddable SPA)
- ✅ NetBox becomes source of truth, no data duplication

**Cons:**
- ❌ Requires iframe/embedding (potential CSP issues)
- ❌ Two-technology stack (Python + TypeScript/Svelte)
- ❌ Rackula UI is separate from NetBox's UI theme
- ❌ Some Rackula features may not make sense in NetBox context (save/load)

**Technical Challenges:**
- **Content Security Policy (CSP):** NetBox has strict CSP; may need to adjust for iframe
- **Authentication:** Rackula needs to authenticate API requests to NetBox
- **State Management:** Rackula expects file-based storage; needs adaptation for live API
- **Device Images:** Need to decide on image storage (NetBox's media files vs. bundled)

### 3.2 Option B: Standalone NetBox Plugin (Native Rewrite)

**Concept:** Build a new rack visualization plugin from scratch using Django templates and NetBox models.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                      NetBox Instance                         │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  NetBox Rack Visualizer Plugin (Django)                │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │  Views (Python)                                   │  │  │
│  │  │  - RackVisualizationView                          │  │  │
│  │  │  - DevicePlacementView                            │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │  Templates (Django + Vanilla JS/Alpine.js)       │  │  │
│  │  │  - SVG rack rendering                             │  │  │
│  │  │  - Device drag-and-drop (vanilla JS)              │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │  API (Django REST Framework)                      │  │  │
│  │  │  - Device placement API                           │  │  │
│  │  │  - Export API (PNG/PDF generation)                │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Implementation Steps:**

1. **Plugin Scaffold (Week 1)**
   - NetBox plugin boilerplate
   - Navigation integration

2. **Rack Visualization Core (Week 2-4)**
   - Django template with SVG rendering
   - Query NetBox Rack + Device models
   - Calculate device positions and render SVG
   - Implement front/rear view toggle

3. **Interactivity (Week 5-7)**
   - JavaScript for drag-and-drop (vanilla or Alpine.js)
   - AJAX endpoints for device placement
   - Collision detection
   - Undo/redo

4. **Advanced Features (Week 8-10)**
   - Device images (load from NetBox media storage)
   - Export to PNG/PDF (Python libraries: Pillow, ReportLab)
   - Airflow visualization
   - Multi-rack support

5. **Polish & Testing (Week 11-12)**
   - NetBox theme integration
   - Permissions
   - Documentation

**Pros:**
- ✅ Native NetBox integration (same UI theme, navigation)
- ✅ Single technology stack (Python/Django)
- ✅ Better permission control (Django ORM-level)
- ✅ Can leverage NetBox's existing UI components

**Cons:**
- ❌ Longer development time (8-12 weeks for feature parity)
- ❌ Need to reimplement Rackula's visualization logic
- ❌ JavaScript drag-and-drop complexity
- ❌ Export to image/PDF requires Python image libraries (more limited than browser Canvas API)

**Technical Challenges:**
- **SVG Rendering:** Need to reimplement Rackula's device rendering logic
- **Drag-and-Drop:** Complex to build from scratch (collision, ghosting, snapping)
- **Image Export:** Server-side image generation is less flexible than client-side Canvas
- **Feature Parity:** Would take significant time to match Rackula's existing features

---

## 4. Data Model Mapping

### 4.1 NetBox → Rackula Field Mapping

The mapping is **very straightforward** because Rackula was designed with NetBox compatibility:

| NetBox Model         | NetBox Field     | Rackula Model | Rackula Field    | Notes                          |
| -------------------- | ---------------- | ------------- | ---------------- | ------------------------------ |
| **Rack**             | name             | Rack          | name             | Direct 1:1                     |
|                      | u_height         | Rack          | height           | Same meaning                   |
|                      | width            | Rack          | width            | Inches (10, 19, 23)            |
|                      | desc_units       | Rack          | desc_units       | U1 at top vs. bottom           |
|                      | type (choices)   | Rack          | form_factor      | Map types to form factors      |
| **Device**           | name             | PlacedDevice  | name             | Direct 1:1                     |
|                      | position         | PlacedDevice  | position         | U position (1-indexed)         |
|                      | face (choices)   | PlacedDevice  | face             | 'front', 'rear', 'both' (TBD)  |
|                      | device_type (FK) | PlacedDevice  | device_type      | References DeviceType.slug     |
| **DeviceType**       | manufacturer (FK)| DeviceType    | manufacturer     | String in Rackula              |
|                      | model            | DeviceType    | model            | Direct 1:1                     |
|                      | slug             | DeviceType    | slug             | Direct 1:1                     |
|                      | u_height         | DeviceType    | u_height         | Direct 1:1                     |
|                      | is_full_depth    | DeviceType    | is_full_depth    | Direct 1:1                     |
|                      | weight           | DeviceType    | weight           | Direct 1:1                     |
|                      | weight_unit      | DeviceType    | weight_unit      | 'kg' or 'lb'                   |
|                      | airflow          | DeviceType    | airflow          | Enum values match              |
|                      | front_image      | DeviceType    | front_image      | Boolean flag                   |
|                      | rear_image       | DeviceType    | rear_image       | Boolean flag                   |
| **Interface**        | name             | InterfaceTemplate | name         | Port name                      |
|                      | type (choices)   | InterfaceTemplate | type         | Interface type enum            |
| **Cable**            | termination_a    | Connection    | a_port_id        | NetBox uses generic FK         |
|                      | termination_b    | Connection    | b_port_id        |                                |

**Key Differences:**

1. **Device Face Field:**
   - **NetBox:** `Device.face` has choices: `front`, `rear` (two-choice field)
   - **Rackula:** `PlacedDevice.face` supports `front`, `rear`, `both` (three-choice)
   - **Resolution:** For full-depth devices in NetBox with no face specified, export as `both` to Rackula

2. **Manufacturer Storage:**
   - **NetBox:** ForeignKey to `Manufacturer` model
   - **Rackula:** String field `DeviceType.manufacturer`
   - **Resolution:** Serialize manufacturer name when exporting to Rackula

3. **Device Position:**
   - Both use 1-indexed U position (U1 is bottom by default)
   - NetBox uses `Device.position` (integer)
   - Rackula uses `PlacedDevice.position` (integer)
   - **Direct mapping, no conversion needed**

4. **Rack Width:**
   - NetBox stores rack type/width (more granular choices)
   - Rackula supports specific inch widths (10, 19, 21, 23)
   - **Resolution:** Map NetBox rack types to nearest Rackula width

5. **Device Colors:**
   - NetBox devices have no color field
   - Rackula assigns category-based colors
   - **Resolution:** Apply Rackula's category color mapping during export

### 4.2 Example Translation Code

```python
# Python (Django plugin view)
from dcim.models import Rack, Device, DeviceType
from netbox_rackula.serializers import RackulaLayoutSerializer

def get_rackula_layout(rack_id):
    """Convert NetBox Rack to Rackula Layout JSON"""
    rack = Rack.objects.get(pk=rack_id)
    devices = Device.objects.filter(rack=rack).select_related('device_type', 'device_type__manufacturer')
    
    # Serialize to Rackula format
    layout = {
        'version': '1.0.0',
        'name': f'{rack.site.name} - {rack.name}' if rack.site else rack.name,
        'racks': [{
            'id': str(rack.pk),
            'name': rack.name,
            'height': rack.u_height,
            'width': map_rack_width(rack.type),
            'desc_units': rack.desc_units,
            'form_factor': map_rack_type(rack.type),
            'show_rear': True,
            'devices': [
                {
                    'id': str(device.pk),
                    'device_type': device.device_type.slug,
                    'position': device.position,
                    'face': map_device_face(device),
                    'name': device.name,
                    'ports': [],  # Populate from device.interfaces
                }
                for device in devices
            ]
        }],
        'device_types': get_device_types(devices),
        'settings': {
            'display_mode': 'label',
            'show_labels_on_images': False,
        }
    }
    
    return layout

def map_device_face(device):
    """Map NetBox device face to Rackula face"""
    if not device.face:
        # No face specified - check if full depth
        if device.device_type.is_full_depth:
            return 'both'
        return 'front'  # Default for half-depth
    return device.face  # 'front' or 'rear'

def map_rack_width(rack_type):
    """Map NetBox rack type to Rackula width"""
    # NetBox types: 2-post-frame, 4-post-frame, 4-post-cabinet, wall-mounted, etc.
    if '10' in rack_type.lower():
        return 10
    elif '23' in rack_type.lower():
        return 23
    return 19  # Default to 19-inch

def get_device_types(devices):
    """Extract unique device types from devices"""
    device_types = {}
    for device in devices:
        dt = device.device_type
        if dt.slug not in device_types:
            device_types[dt.slug] = {
                'slug': dt.slug,
                'manufacturer': dt.manufacturer.name if dt.manufacturer else None,
                'model': dt.model,
                'u_height': dt.u_height,
                'is_full_depth': dt.is_full_depth,
                'weight': dt.weight,
                'weight_unit': dt.weight_unit,
                'airflow': dt.airflow,
                'colour': get_category_color(dt),
                'category': infer_category(dt),
                'front_image': dt.front_image,
                'rear_image': dt.rear_image,
            }
    return list(device_types.values())
```

---

## 5. Recommended Approach: Option A (Detailed Plan)

### 5.1 Phase 1: Plugin Foundation (Week 1)

**Goals:**
- NetBox plugin scaffold
- Basic navigation integration
- Proof-of-concept view

**Tasks:**
1. Create plugin repository `netbox-plugin-rackula`
2. Implement `setup.py` with NetBox plugin metadata
3. Register navigation item: "Rack Visualizer" under DCIM menu
4. Create basic Django view: `RackVisualizerView`
5. Template that displays "Rackula Visualizer" heading
6. Install and test plugin in development NetBox instance

**Deliverables:**
- Plugin installs without errors
- Navigation item appears in DCIM menu
- Clicking opens placeholder view

### 5.2 Phase 2: Data Translation API (Week 2)

**Goals:**
- REST API endpoint that exports NetBox rack data to Rackula JSON format
- Handles device types, devices, rack properties

**Tasks:**
1. Create API view: `/api/plugins/rackula/rack/<id>/layout/`
2. Implement `RackulaLayoutSerializer`
   - Query NetBox: `Rack`, `Device`, `DeviceType`, `Interface`
   - Map to Rackula JSON schema
   - Handle device face mapping (full-depth → both)
   - Apply category colors (use Rackula's constants)
3. Add optional query params:
   - `?include_images=true` - Include device image URLs
   - `?include_cables=true` - Include cable/connection data
4. Write unit tests for serialization
5. Document API endpoint (OpenAPI schema)

**Example Response:**
```json
{
  "version": "1.0.0",
  "name": "DC1-R42",
  "racks": [{
    "id": "123",
    "name": "R42",
    "height": 42,
    "width": 19,
    "devices": [...]
  }],
  "device_types": [...],
  "settings": {...}
}
```

### 5.3 Phase 3: Embed Rackula UI (Week 3)

**Goals:**
- Build Rackula as embeddable static bundle
- Serve from Django plugin
- Pass rack ID to Rackula via URL/data attributes

**Tasks:**
1. **Modify Rackula Build:**
   - Create `vite.config.embed.ts` for embeddable mode
   - Single HTML file with inlined JS/CSS (or small set of files)
   - Accept configuration via `data-*` attributes or `window` global
   - Read rack data from API endpoint instead of localStorage

2. **Plugin Static Files:**
   - Copy Rackula build to `netbox_rackula/static/rackula/`
   - Serve via Django's static file handler

3. **Django Template Integration:**
   ```django
   {# templates/netbox_rackula/rack_visualizer.html #}
   {% extends 'base.html' %}
   {% load static %}

   {% block content %}
     <div id="rackula-container" 
          data-api-url="/api/plugins/rackula/rack/{{ rack.pk }}/layout/"
          data-rack-id="{{ rack.pk }}">
     </div>
     <script src="{% static 'rackula/rackula.js' %}"></script>
   {% endblock %}
   ```

4. **Test End-to-End:**
   - Navigate to rack in NetBox
   - Click "Visualize" button
   - Rackula loads with NetBox data

### 5.4 Phase 4: Bidirectional Sync (Week 4-5) - OPTIONAL FOR MVP

**Goals:**
- Allow users to modify rack layout in Rackula
- Save changes back to NetBox

**Tasks:**
1. Add POST endpoint: `/api/plugins/rackula/rack/<id>/layout/`
2. Validate incoming Rackula JSON
3. Update NetBox models:
   - `Device.position` (move devices)
   - `Device.face` (change mounting face)
   - Create new devices if needed
4. Handle conflicts and validation errors
5. Transaction safety (rollback on error)

**Challenges:**
- Device creation: requires device role, site assignment
- User permissions: check NetBox RBAC before allowing changes
- Concurrent edits: handle stale data

**Decision Point:** For MVP, consider making this **read-only**. Users view in Rackula, edit in NetBox UI.

### 5.5 Phase 5: Polish & Deployment (Week 6)

**Goals:**
- Production-ready plugin
- Documentation
- Packaging for PyPI

**Tasks:**
1. **Error Handling:**
   - Handle missing racks, empty racks
   - API error messages
   - Graceful degradation if Rackula fails to load

2. **Permissions:**
   - Check `dcim.view_rack` permission
   - Check `dcim.change_device` if write mode enabled

3. **Documentation:**
   - README with installation instructions
   - Configuration options
   - Screenshots

4. **Testing:**
   - Unit tests for serializers
   - Integration tests with test NetBox instance
   - Selenium/Playwright tests for UI

5. **Package:**
   - `setup.py` with dependencies
   - Publish to PyPI: `pip install netbox-plugin-rackula`

### 5.6 MVP Feature Set

**Included in MVP:**
- ✅ View NetBox rack layout in Rackula UI
- ✅ Front/rear rack views
- ✅ Device images (if available in NetBox)
- ✅ Export to PNG/PDF from Rackula
- ✅ Airflow visualization
- ✅ Device details tooltip

**Excluded from MVP (Future Enhancements):**
- ❌ Drag-and-drop editing (read-only MVP)
- ❌ Multi-rack selection
- ❌ Cable visualization (phase 2)
- ❌ Custom device images upload (use NetBox's image fields)

---

## 6. Technical Considerations

### 6.1 Authentication & Authorization

**Challenge:** Rackula (client-side JS) needs to authenticate with NetBox API.

**Solution:**
- Use Django's session authentication (user already logged into NetBox)
- Rackula API endpoints inherit NetBox's auth middleware
- CSRF token handling for POST requests
- Check permissions: `dcim.view_rack`, `dcim.view_device`

### 6.2 Device Images

**Challenge:** Rackula expects device images as WebP files in specific paths. NetBox stores images as media files.

**Options:**

1. **Use NetBox's Image Fields:**
   - NetBox models have `front_image` and `rear_image` fields (FileField)
   - Plugin serves images from NetBox media storage
   - Rackula loads images via URL passed in API response

2. **Bundle Images in Plugin:**
   - Include Rackula's existing device image library
   - Serve from plugin static files
   - Match by device type slug

**Recommendation:** Option 1 (use NetBox's images) for true NetBox integration.

### 6.3 Content Security Policy (CSP)

**Challenge:** NetBox has strict CSP. Embedding Rackula (if in iframe) or inline scripts may be blocked.

**Solutions:**
- **Option 1:** Adjust NetBox CSP to allow plugin static files
- **Option 2:** Build Rackula without `eval()` or inline scripts (strict CSP mode)
- **Option 3:** Use Django template rendering instead of iframe

**Recommendation:** Option 2 or 3. Avoid iframe if possible.

### 6.4 Rack Width Mapping

**Challenge:** NetBox rack types are more granular than Rackula's inch widths.

**Mapping Table:**

| NetBox Rack Type     | Rackula Width |
| -------------------- | ------------- |
| 2-post-frame         | 19            |
| 4-post-frame         | 19            |
| 4-post-cabinet       | 19            |
| wall-mounted         | 10            |
| wall-mounted-vertical| 10            |
| (custom types)       | 19 (default)  |

### 6.5 Device Face Handling

**NetBox Device Face Choices:**
```python
# NetBox: dcim/models/devices.py
DEVICE_FACE_FRONT = 'front'
DEVICE_FACE_REAR = 'rear'
```

**Rackula Device Face:**
```typescript
type DeviceFace = 'front' | 'rear' | 'both';
```

**Mapping Logic:**
- NetBox `face='front'` → Rackula `face='front'`
- NetBox `face='rear'` → Rackula `face='rear'`
- NetBox `face=None` and `is_full_depth=True` → Rackula `face='both'`
- NetBox `face=None` and `is_full_depth=False` → Rackula `face='front'`

### 6.6 Cable/Connection Visualization

**Current State:**
- Rackula has MVP port-to-port connection tracking
- NetBox has robust Cable model with terminations

**Integration Path:**
1. Query NetBox `Cable` objects for devices in rack
2. Map cable terminations to Rackula `Connection` objects
3. Use Rackula's connection visualization (SVG lines between ports)

**Challenge:** Rackula's connection model is simpler than NetBox's. May need to simplify NetBox cables for display.

---

## 7. Alternative Approach: Read-Only Visualization

### 7.1 Simplified MVP (2-3 Weeks)

If full plugin development is too much effort, consider a **read-only visualization plugin**:

**Scope:**
- No drag-and-drop editing
- No save-back to NetBox
- Just visualization + export

**Benefits:**
- Faster development (2-3 weeks)
- Lower risk
- Still provides value (better visualization than NetBox's default rack view)

**Implementation:**
- Same architecture as Option A
- Skip bidirectional sync (Phase 4)
- Rackula loads NetBox data, displays, exports
- Users edit in NetBox UI, visualize in plugin

---

## 8. Estimated Development Effort

### Option A: Plugin with Rackula Integration

| Phase                     | Duration | Tasks                                              |
| ------------------------- | -------- | -------------------------------------------------- |
| 1. Plugin Foundation      | 1 week   | Scaffold, navigation, basic view                   |
| 2. Data Translation API   | 1 week   | Serializer, API endpoint, testing                  |
| 3. Embed Rackula UI       | 1 week   | Build modifications, static serving, integration   |
| 4. Bidirectional Sync     | 2 weeks  | POST endpoint, validation, NetBox updates          |
| 5. Polish & Deployment    | 1 week   | Error handling, permissions, docs, packaging       |
| **Total (MVP + Sync)**    | **6 weeks** | Full read-write plugin                          |
| **Total (Read-Only MVP)** | **3 weeks** | Visualization + export only                     |

### Option B: Standalone NetBox Plugin

| Phase                          | Duration | Tasks                                              |
| ------------------------------ | -------- | -------------------------------------------------- |
| 1. Plugin Foundation           | 1 week   | Scaffold, navigation                               |
| 2. Rack Visualization Core     | 3 weeks  | SVG rendering, device layout, views                |
| 3. Interactivity               | 3 weeks  | Drag-and-drop, collision, API endpoints            |
| 4. Advanced Features           | 3 weeks  | Images, export, airflow, multi-rack                |
| 5. Polish & Deployment         | 2 weeks  | Testing, docs, packaging                           |
| **Total (Feature Parity)**     | **12 weeks** | Full reimplementation                          |

---

## 9. Recommendations

### 9.1 Recommended Path: Option A (Read-Only MVP)

**Why:**
- **Fastest time to value:** 3 weeks for read-only, 6 weeks for read-write
- **Reuses proven code:** Leverages Rackula's existing visualization
- **Lower risk:** Minimal changes to both Rackula and NetBox
- **Incremental:** Can start with read-only, add write-back later

**Suggested Phases:**
1. **Phase 1 (Week 1-3):** Read-only visualization plugin
   - View NetBox racks in Rackula UI
   - Export to PNG/PDF
   - Airflow and device images

2. **Phase 2 (Week 4-6) - Optional:** Write-back capability
   - Drag-and-drop editing
   - Save changes to NetBox

3. **Phase 3 (Future):** Advanced features
   - Cable visualization
   - Multi-rack editing
   - Rack comparison views

### 9.2 Success Criteria

**MVP is successful if:**
- ✅ Users can view any NetBox rack in enhanced Rackula visualization
- ✅ Device images display correctly (from NetBox or bundled library)
- ✅ Export to PNG/PDF works
- ✅ Plugin installs cleanly on NetBox 4.x
- ✅ No performance issues with 100+ device racks

### 9.3 Risks & Mitigations

| Risk                                  | Impact | Mitigation                                           |
| ------------------------------------- | ------ | ---------------------------------------------------- |
| NetBox CSP blocks Rackula scripts     | High   | Build Rackula in strict CSP mode, avoid inline JS   |
| Device images missing in NetBox       | Medium | Fallback to bundled images, show colored rectangles |
| Performance with large racks          | Medium | Lazy loading, pagination, optimize API queries       |
| Rackula build too large               | Low    | Code splitting, tree shaking, minification           |
| NetBox version compatibility          | Low    | Test on NetBox LTS releases (4.0, 4.2)               |

---

## 10. Next Steps

### 10.1 Immediate Actions

1. **Validate Assumptions:**
   - Confirm NetBox version(s) to support (4.0+ recommended)
   - Verify CSP requirements in target NetBox deployment
   - Check if device images are available in NetBox instance

2. **Proof of Concept:**
   - Build minimal plugin that displays "Hello from Rackula Plugin"
   - Test Rackula build in embeddable mode (single HTML file)
   - Verify API authentication works with NetBox session

3. **Stakeholder Review:**
   - Present this analysis to stakeholders
   - Confirm scope: read-only MVP vs. full read-write
   - Decide on timeline and resource allocation

### 10.2 Development Roadmap (if approved)

**Month 1:**
- Weeks 1-3: Read-only visualization MVP
- Week 4: Internal testing and feedback

**Month 2:**
- Weeks 5-6: Write-back capability (if required)
- Weeks 7-8: Beta testing with users

**Month 3+:**
- Advanced features (cable viz, multi-rack, etc.)
- Production deployment
- Maintenance and support

### 10.3 Resources Required

**Development Team:**
- 1x Full-stack developer (Python/Django + TypeScript/Svelte)
- Part-time support: DevOps for deployment, QA for testing

**Infrastructure:**
- Development NetBox instance (Docker or VM)
- Test rack data (seed database with sample racks/devices)
- PyPI account (for package publishing)

---

## 11. Conclusion

**Yes, converting Rackula to a NetBox plugin is highly feasible and practical.**

The key insight is that Rackula was **already designed with NetBox compatibility in mind**:
- Field naming conventions match NetBox
- Data model mirrors NetBox's rack/device structure
- Device library sourced from NetBox community library
- Existing import scripts prove integration is possible

**The recommended approach** (Option A: Plugin with Rackula Integration) offers:
- ✅ Fastest development: 3 weeks for read-only MVP
- ✅ Best of both worlds: NetBox's data management + Rackula's visualization
- ✅ Low risk: Minimal changes to either system
- ✅ Incremental: Can start small, expand features later

**Next Decision Point:** Confirm scope (read-only vs. read-write) and timeline with stakeholders.

---

## Appendices

### A. Rackula File Manifest (Relevant Files)

**Data Models:**
- `src/lib/types/index.ts` - Core type definitions (NetBox-compatible)
- `src/lib/types/constants.ts` - Category colors and constants

**Stores (State Management):**
- `src/lib/stores/layout.svelte.ts` - Rack and device state

**NetBox Integration:**
- `scripts/import-netbox-devices.ts` - Import single devices
- `scripts/bulk-import-netbox.ts` - Bulk import from NetBox library

**Components:**
- `src/lib/components/Rack.svelte` - Main rack visualization
- `src/lib/components/RackDevice.svelte` - Device rendering

### B. NetBox Plugin Resources

**Official Documentation:**
- Plugin Development Guide: https://docs.netbox.dev/en/stable/plugins/development/
- API Documentation: https://docs.netbox.dev/en/stable/integrations/rest-api/

**Example Plugins:**
- netbox-topology-views: https://github.com/netbox-community/netbox-topology-views
- netbox-plugin-device-map: https://github.com/netbox-community/netbox-plugin-device-map

**Community:**
- NetBox GitHub: https://github.com/netbox-community/netbox
- NetBox Slack: https://netdev.chat/

### C. Technology Stack Comparison

| Aspect              | Rackula                      | NetBox Plugin                |
| ------------------- | ---------------------------- | ---------------------------- |
| Language            | TypeScript                   | Python                       |
| Framework           | Svelte 5                     | Django 4.2+                  |
| UI                  | Custom SVG components        | Django templates             |
| State Management    | Svelte runes                 | Django ORM                   |
| Data Storage        | Client-side (browser)        | PostgreSQL                   |
| Auth                | N/A (standalone)             | Django auth + NetBox RBAC    |
| Deployment          | Static site (nginx/GitHub)   | Python package (pip install) |
| API                 | N/A (no backend)             | Django REST Framework        |

### D. Sample NetBox API Response

**Endpoint:** `GET /api/dcim/racks/42/`

```json
{
  "id": 42,
  "url": "https://netbox.example.com/api/dcim/racks/42/",
  "display": "R42",
  "name": "R42",
  "facility_id": null,
  "site": {
    "id": 1,
    "url": "https://netbox.example.com/api/dcim/sites/1/",
    "display": "DC1",
    "name": "DC1",
    "slug": "dc1"
  },
  "location": null,
  "status": {
    "value": "active",
    "label": "Active"
  },
  "role": null,
  "tenant": null,
  "serial": "",
  "asset_tag": null,
  "type": {
    "value": "4-post-cabinet",
    "label": "4-post cabinet"
  },
  "width": {
    "value": 19,
    "label": "19 inches"
  },
  "u_height": 42,
  "desc_units": false,
  "outer_width": null,
  "outer_depth": null,
  "outer_unit": null,
  "mounting_depth": null,
  "weight": null,
  "max_weight": null,
  "weight_unit": null,
  "device_count": 15,
  "comments": "",
  "tags": [],
  "custom_fields": {},
  "created": "2023-01-15T10:30:00.000000Z",
  "last_updated": "2023-12-20T14:45:00.000000Z"
}
```

**Endpoint:** `GET /api/dcim/devices/?rack_id=42`

```json
{
  "count": 15,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 123,
      "url": "https://netbox.example.com/api/dcim/devices/123/",
      "display": "server01",
      "name": "server01",
      "device_type": {
        "id": 45,
        "url": "https://netbox.example.com/api/dcim/device-types/45/",
        "display": "Dell PowerEdge R650",
        "manufacturer": {
          "id": 1,
          "url": "https://netbox.example.com/api/dcim/manufacturers/1/",
          "display": "Dell",
          "name": "Dell",
          "slug": "dell"
        },
        "model": "PowerEdge R650",
        "slug": "poweredge-r650"
      },
      "device_role": {
        "id": 3,
        "url": "https://netbox.example.com/api/dcim/device-roles/3/",
        "display": "Server",
        "name": "Server",
        "slug": "server"
      },
      "face": {
        "value": "front",
        "label": "Front"
      },
      "position": 40,
      "rack": 42,
      "site": 1,
      "location": null,
      "status": {
        "value": "active",
        "label": "Active"
      },
      "airflow": {
        "value": "front-to-rear",
        "label": "Front to rear"
      },
      "primary_ip": null,
      "serial": "",
      "asset_tag": null,
      "comments": "",
      "tags": [],
      "custom_fields": {}
    }
  ]
}
```

---

**End of Analysis**

*This document provides a comprehensive analysis of NetBox plugin conversion feasibility for the Rackula project. For questions or clarifications, please reach out to the development team.*
