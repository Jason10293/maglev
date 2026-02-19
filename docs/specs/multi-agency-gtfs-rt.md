# Multi-Agency GTFS-RT Feeds Specification

## Problem Statement

Maglev currently supports loading static GTFS data containing multiple agencies (via a merged `agencies.txt`), but only consumes the **first** configured GTFS-RT feed. This means real-time vehicle positions, trip updates, and service alerts are limited to a single agency's data, even when the static dataset covers several agencies.

### Current Limitations

1. **`gtfs.Config`** has single-valued fields (`TripUpdatesURL`, `VehiclePositionsURL`, `ServiceAlertsURL`, plus one auth header pair), so only one feed source is representable.
2. **`appconf.JSONConfig.ToGtfsConfigData()`** explicitly extracts `GtfsRtFeeds[0]` and discards the rest.
3. **`Manager.updateGTFSRealtime()`** fetches from three fixed URLs and **replaces** the in-memory slices on each cycle - there is no merge logic.
4. **No agency mapping** exists between RT feed entities and static GTFS agencies. Vehicle/trip matching relies on route IDs in the RT data matching route IDs in the static data, which works only when both come from the same source.

## Goals

- Support **N** independent GTFS-RT feed sources, each with its own URLs and auth credentials.
- Support **vehicle positions**, **trip updates**, and **service alerts** per feed.
- Provide an optional **agency mapping** per feed that maps the RT feed onto a specific agency (or agencies) in the static GTFS `agencies.txt`.
- Merge data from all feeds into the existing in-memory data structures so that downstream handlers and API consumers require **no changes**.
- Maintain backward compatibility with existing single-feed configurations.

## Non-Goals

- Per-feed polling intervals (all feeds share the existing 30-second cycle).
- Per-feed health status or circuit breakers (can be added later).
- Transforming or rewriting IDs within the protobuf data itself.

---

## Configuration

### JSON Config Schema

The existing `gtfs-rt-feeds` array is already defined in `config.schema.json`. Each feed object gains one new optional field: `agency-id`.

```json
{
  "gtfs-rt-feeds": [
    {
      "trip-updates-url": "https://agency-a.example.com/trip-updates.pb",
      "vehicle-positions-url": "https://agency-a.example.com/vehicle-positions.pb",
      "service-alerts-url": "https://agency-a.example.com/alerts.pb",
      "realtime-auth-header-name": "Authorization",
      "realtime-auth-header-value": "Bearer token-a",
      "agency-id": "agency_a"
    },
    {
      "trip-updates-url": "https://agency-b.example.com/trip-updates.pb",
      "vehicle-positions-url": "https://agency-b.example.com/vehicle-positions.pb",
      "service-alerts-url": "",
      "realtime-auth-header-name": "",
      "realtime-auth-header-value": "",
      "agency-id": "agency_b"
    },
    {
      "trip-updates-url": "https://regional.example.com/trip-updates.pb",
      "vehicle-positions-url": "https://regional.example.com/vehicle-positions.pb",
      "service-alerts-url": "https://regional.example.com/alerts.pb",
      "realtime-auth-header-name": "",
      "realtime-auth-header-value": ""
    }
  ]
}
```

### `agency-id` Field Semantics

| Scenario | Behavior |
|---|---|
| `agency-id` is **omitted or empty** | The feed is treated as a "global" feed. All entities are included as-is. Vehicle/trip matching uses the existing route-ID-based matching logic against all agencies. This preserves current behavior for single-feed setups. |
| `agency-id` is **set** (e.g., `"agency_a"`) | The feed is scoped to that agency. During the merge phase, the agency ID is used for two purposes: (1) **validation** - entities whose route IDs don't belong to the mapped agency are logged as warnings and skipped, and (2) **conflict resolution** - if two feeds provide data for the same trip or vehicle ID, the feed with an explicit agency mapping takes precedence over a global feed. |

The `agency-id` value must match an `agency_id` in the static GTFS `agencies.txt`. If it doesn't match any loaded agency, a warning is logged at startup and that feed's data is still loaded (to handle cases where agencies are added to the static feed after configuration).

### Config Schema Changes (`config.schema.json`)

Add `agency-id` to the feed object properties:

```json
{
  "agency-id": {
    "type": "string",
    "description": "Optional agency ID from agencies.txt that this RT feed maps to. When set, entities from this feed are scoped to the specified agency. When omitted, entities are treated as global."
  }
}
```

### Environment Variable Overrides

The existing `GTFS_REALTIME_AUTH_NAME` and `GTFS_REALTIME_AUTH_VALUE` environment variables continue to override only the **first** feed's auth headers. No new per-feed environment variables are added in this iteration; operators needing per-feed overrides should use the JSON config.

### Command-Line Flags

The existing `--trip-updates-url`, `--vehicle-positions-url`, `--service-alerts-url`, and related flags continue to configure a **single** feed. Multi-feed configuration requires the JSON config file (`-f`). This is consistent with the existing mutual exclusivity between `-f` and other flags.

---

## Internal Data Model Changes

### `appconf.GtfsRtFeed` (existing struct, add field)

```go
type GtfsRtFeed struct {
    TripUpdatesURL          string `json:"trip-updates-url"`
    VehiclePositionsURL     string `json:"vehicle-positions-url"`
    ServiceAlertsURL        string `json:"service-alerts-url"`
    RealTimeAuthHeaderName  string `json:"realtime-auth-header-name"`
    RealTimeAuthHeaderValue string `json:"realtime-auth-header-value"`
    AgencyID                string `json:"agency-id"`           // NEW
}
```

### `appconf.GtfsConfigData` (replace single-feed fields with slice)

```go
type GtfsConfigData struct {
    GtfsURL               string
    StaticAuthHeaderKey    string
    StaticAuthHeaderValue  string
    RtFeeds               []RtFeedConfig   // NEW: replaces single-feed fields
    GTFSDataPath           string
    Env                    Environment
    Verbose                bool
    EnableGTFSTidy         bool
}

// RtFeedConfig holds the configuration for a single GTFS-RT feed source.
type RtFeedConfig struct {
    TripUpdatesURL      string
    VehiclePositionsURL string
    ServiceAlertsURL    string
    AuthHeaderKey       string
    AuthHeaderValue     string
    AgencyID            string   // empty = global feed
}
```

For backward compatibility, `ToGtfsConfigData()` iterates **all** feeds instead of just the first:

```go
func (j *JSONConfig) ToGtfsConfigData() GtfsConfigData {
    cfg := GtfsConfigData{
        GtfsURL:               j.GtfsStaticFeed.URL,
        StaticAuthHeaderKey:   j.GtfsStaticFeed.AuthHeaderName,
        StaticAuthHeaderValue: j.GtfsStaticFeed.AuthHeaderValue,
        GTFSDataPath:          j.DataPath,
        Env:                   EnvFlagToEnvironment(j.Env),
        Verbose:               true,
        EnableGTFSTidy:        j.GtfsStaticFeed.EnableGTFSTidy,
    }

    for _, feed := range j.GtfsRtFeeds {
        cfg.RtFeeds = append(cfg.RtFeeds, RtFeedConfig{
            TripUpdatesURL:      feed.TripUpdatesURL,
            VehiclePositionsURL: feed.VehiclePositionsURL,
            ServiceAlertsURL:    feed.ServiceAlertsURL,
            AuthHeaderKey:       feed.RealTimeAuthHeaderName,
            AuthHeaderValue:     feed.RealTimeAuthHeaderValue,
            AgencyID:            feed.AgencyID,
        })
    }

    return cfg
}
```

### `gtfs.Config` (replace single-feed fields with slice)

```go
type Config struct {
    GtfsURL               string
    StaticAuthHeaderKey    string
    StaticAuthHeaderValue  string
    RtFeeds               []RtFeedConfig   // NEW: replaces single-feed fields
    GTFSDataPath           string
    Env                    appconf.Environment
    Verbose                bool
    EnableGTFSTidy         bool
}

// RtFeedConfig is re-exported from appconf or defined here to avoid import cycles.
type RtFeedConfig struct {
    TripUpdatesURL      string
    VehiclePositionsURL string
    ServiceAlertsURL    string
    AuthHeaderKey       string
    AuthHeaderValue     string
    AgencyID            string
}

func (config Config) realTimeDataEnabled() bool {
    for _, feed := range config.RtFeeds {
        if feed.TripUpdatesURL != "" || feed.VehiclePositionsURL != "" {
            return true
        }
    }
    return false
}
```

### `cmd/api/main.go` Conversion

When using the JSON config path, the conversion from `GtfsConfigData` to `gtfs.Config` copies the `RtFeeds` slice directly. When using CLI flags, a single `RtFeedConfig` is constructed from the flag values, preserving backward compatibility:

```go
// CLI flag path (no -f):
gtfsCfg.RtFeeds = []gtfs.RtFeedConfig{{
    TripUpdatesURL:      tripUpdatesURL,
    VehiclePositionsURL: vehiclePositionsURL,
    ServiceAlertsURL:    serviceAlertsURL,
    AuthHeaderKey:       rtAuthHeaderKey,
    AuthHeaderValue:     rtAuthHeaderValue,
}}
```

---

## Feed Fetching

### `Manager.updateGTFSRealtime()` Changes

The method currently fetches from three fixed URLs. It will be refactored to iterate over all configured feeds and merge the results.

#### Pseudocode

```go
func (manager *Manager) updateGTFSRealtime(ctx context.Context, config Config) {
    logger := logging.FromContext(ctx).With("component", "gtfs_realtime")

    type feedResult struct {
        feedIndex int
        agencyID  string
        trips     []gtfs.Trip
        vehicles  []gtfs.Vehicle
        alerts    []gtfs.Alert
    }

    results := make([]feedResult, len(config.RtFeeds))
    var wg sync.WaitGroup

    // Fetch all feeds in parallel
    for i, feed := range config.RtFeeds {
        wg.Add(1)
        go func(idx int, f RtFeedConfig) {
            defer wg.Done()

            headers := buildAuthHeaders(f)
            result := feedResult{feedIndex: idx, agencyID: f.AgencyID}

            // Fetch each feed type in parallel within this feed
            var innerWg sync.WaitGroup

            if f.TripUpdatesURL != "" {
                innerWg.Add(1)
                go func() {
                    defer innerWg.Done()
                    data, err := loadRealtimeData(ctx, f.TripUpdatesURL, headers)
                    if err != nil {
                        logger.Error("trip updates fetch failed", "feed", idx, "err", err)
                        return
                    }
                    result.trips = data.Trips
                }()
            }

            if f.VehiclePositionsURL != "" {
                innerWg.Add(1)
                go func() {
                    defer innerWg.Done()
                    data, err := loadRealtimeData(ctx, f.VehiclePositionsURL, headers)
                    if err != nil {
                        logger.Error("vehicle positions fetch failed", "feed", idx, "err", err)
                        return
                    }
                    result.vehicles = data.Vehicles
                }()
            }

            if f.ServiceAlertsURL != "" {
                innerWg.Add(1)
                go func() {
                    defer innerWg.Done()
                    data, err := loadRealtimeData(ctx, f.ServiceAlertsURL, headers)
                    if err != nil {
                        logger.Error("service alerts fetch failed", "feed", idx, "err", err)
                        return
                    }
                    result.alerts = data.Alerts
                }()
            }

            innerWg.Wait()
            results[idx] = result
        }(i, feed)
    }

    wg.Wait()

    if ctx.Err() != nil {
        return
    }

    // Merge results
    manager.mergeRealtimeResults(results)
}
```

### Merge Strategy

```go
func (manager *Manager) mergeRealtimeResults(results []feedResult) {
    manager.realTimeMutex.Lock()
    defer manager.realTimeMutex.Unlock()

    // Pre-allocate with estimated capacity
    var totalTrips, totalVehicles, totalAlerts int
    for _, r := range results {
        totalTrips += len(r.trips)
        totalVehicles += len(r.vehicles)
        totalAlerts += len(r.alerts)
    }

    allTrips := make([]gtfs.Trip, 0, totalTrips)
    allVehicles := make([]gtfs.Vehicle, 0, totalVehicles)
    allAlerts := make([]gtfs.Alert, 0, totalAlerts)

    // Track seen IDs for conflict resolution
    seenTrips := make(map[string]int)       // trip ID -> feed index
    seenVehicles := make(map[string]int)    // vehicle ID -> feed index

    for _, r := range results {
        for _, trip := range r.trips {
            tripID := trip.ID.ID
            if prevFeed, exists := seenTrips[tripID]; exists {
                // Agency-scoped feed wins over global feed
                if r.agencyID != "" && results[prevFeed].agencyID == "" {
                    // Replace: find and overwrite the previous entry
                    for i, t := range allTrips {
                        if t.ID.ID == tripID {
                            allTrips[i] = trip
                            break
                        }
                    }
                    seenTrips[tripID] = r.feedIndex
                }
                // Otherwise keep the first occurrence (earlier feeds have priority)
                continue
            }
            allTrips = append(allTrips, trip)
            seenTrips[tripID] = r.feedIndex
        }

        for _, vehicle := range r.vehicles {
            vehicleID := ""
            if vehicle.ID != nil {
                vehicleID = vehicle.ID.ID
            }
            if vehicleID != "" {
                if prevFeed, exists := seenVehicles[vehicleID]; exists {
                    if r.agencyID != "" && results[prevFeed].agencyID == "" {
                        for i, v := range allVehicles {
                            if v.ID != nil && v.ID.ID == vehicleID {
                                allVehicles[i] = vehicle
                                break
                            }
                        }
                        seenVehicles[vehicleID] = r.feedIndex
                    }
                    continue
                }
                seenVehicles[vehicleID] = r.feedIndex
            }
            allVehicles = append(allVehicles, vehicle)
        }

        // Alerts are additive - no deduplication (alerts don't have unique IDs in GTFS-RT)
        allAlerts = append(allAlerts, r.alerts...)
    }

    manager.realTimeTrips = allTrips
    manager.realTimeVehicles = allVehicles
    manager.realTimeAlerts = allAlerts

    filterRealTimeVehicleByValidId(manager)
    rebuildRealTimeTripLookup(manager)
    rebuildRealTimeVehicleLookupByTrip(manager)
    rebuildRealTimeVehicleLookupByVehicle(manager)
}
```

### Conflict Resolution Rules

| Scenario | Resolution |
|---|---|
| Same trip ID from two agency-scoped feeds | First feed in config order wins. Log a warning. |
| Same trip ID from an agency-scoped feed and a global feed | Agency-scoped feed wins. |
| Same trip ID from two global feeds | First feed in config order wins. Log a warning. |
| Same vehicle ID from two feeds | Same rules as trip ID conflicts. |
| Duplicate alerts | No deduplication. Alerts are additive across all feeds. |

---

## Periodic Updates

### `updateGTFSRealtimePeriodically()`

The method signature changes to accept the full `Config` (which it already does) and passes it through to the updated `updateGTFSRealtime()`. No structural change needed here since `Config` now carries `RtFeeds`.

---

## Downstream Handler Impact

The in-memory data structures (`realTimeTrips`, `realTimeVehicles`, `realTimeAlerts` and their lookup maps) remain the same types and serve the same role. All downstream code that accesses real-time data through the `Manager` methods continues to work without modification:

- `GetRealTimeTrips()` / `GetRealTimeVehicles()` - return merged data from all feeds.
- `VehiclesForAgencyID(agencyID)` - continues to filter by route ID, which naturally scopes to the correct agency because different agencies have different route IDs.
- `GetVehicleForTrip(tripID)` - continues to search the merged vehicle list.
- `GetVehicleByID(vehicleID)` - continues O(1) lookup from the merged map.
- `GetTripUpdateByID(tripID)` - continues O(1) lookup from the merged map.
- `GetAlertsForRoute(routeID)` / `GetAlertsByIDs()` / `GetAlertsForTrip()` / `GetAlertsForStop()` - all filter by entity IDs, which work across merged data.

**No REST API handler changes are required.**

---

## Validation

### Startup Validation

When the config is loaded and the GTFS manager is initialized:

1. For each feed with a non-empty `agency-id`, check that the agency ID exists in the loaded static GTFS data (`manager.agenciesMap`).
2. If the agency ID is not found, log a **warning** (not an error) and continue. The feed is still loaded.
3. Validate that each feed has at least one URL configured (`trip-updates-url` or `vehicle-positions-url`). Feeds with no URLs are skipped with a warning.
4. Validate that auth headers are provided in pairs (both name and value, or neither).

### Runtime Logging

Each update cycle logs:
- The number of feeds fetched successfully vs. failed.
- Per-feed entity counts (trips, vehicles, alerts) at debug level.
- Any conflicts detected during merge at warn level.
- Total merged entity counts at info level.

---

## Testing Strategy

### Unit Tests

1. **Config parsing**: Verify that multiple feeds with `agency-id` are parsed correctly from JSON.
2. **Merge logic**: Test `mergeRealtimeResults()` with:
   - Two feeds with no overlapping IDs (simple concatenation).
   - Two feeds with overlapping trip IDs (conflict resolution).
   - Agency-scoped feed vs. global feed conflict.
   - A feed that fails to fetch (partial data merge).
   - Empty feeds (no data returned).
3. **`realTimeDataEnabled()`**: Returns true if any feed has URLs configured.

### Integration Tests

1. **Multi-feed test setup**: Create a test helper similar to `createTestApiWithRealTimeData` that stands up multiple HTTP test servers, each serving different protobuf data, and configures the manager with multiple `RtFeedConfig` entries.
2. **`VehiclesForAgencyID`**: With two feeds for two different agencies, verify that requesting vehicles for agency A returns only agency A's vehicles and vice versa.
3. **Cross-feed alert aggregation**: Verify that alerts from multiple feeds are all returned by `GetAlertsForRoute()`.

### Test Data

The existing `raba-vehicle-positions.pb` and `raba-trip-updates.pb` test data can be used for one feed. A second set of test data will need to be generated or the existing data can be served from two endpoints with different configurations to verify merge behavior.

---

## Migration / Backward Compatibility

### Config File Migration

Existing config files with a single entry in `gtfs-rt-feeds` continue to work without modification. The `agency-id` field is optional and defaults to empty (global feed behavior).

### CLI Flag Migration

CLI flags (`--trip-updates-url`, `--vehicle-positions-url`, etc.) construct a single-entry `RtFeeds` slice, preserving identical behavior to the current implementation.

### `GtfsConfigData` Migration

The old single-feed fields (`TripUpdatesURL`, `VehiclePositionsURL`, etc.) on `GtfsConfigData` are replaced by `RtFeeds`. Code in `cmd/api/main.go` that converts `GtfsConfigData` to `gtfs.Config` is updated to copy the slice. The old fields are removed since `GtfsConfigData` is an internal struct not exposed to external consumers.

---

## Implementation Order

### Phase 1: Config Layer
- Add `AgencyID` field to `appconf.GtfsRtFeed`.
- Add `RtFeedConfig` struct and `RtFeeds` field to `appconf.GtfsConfigData`.
- Update `ToGtfsConfigData()` to iterate all feeds.
- Update `config.schema.json` with `agency-id` property.
- Update `config.example.json` with a second feed example.
- Add config parsing tests.

### Phase 2: GTFS Config Layer
- Add `RtFeedConfig` struct and `RtFeeds` field to `gtfs.Config`.
- Remove old single-feed fields from `gtfs.Config`.
- Update `realTimeDataEnabled()` to check any feed.
- Update `cmd/api/main.go` conversion code for both JSON and CLI paths.

### Phase 3: Feed Fetching and Merge
- Refactor `updateGTFSRealtime()` to iterate all feeds.
- Implement `mergeRealtimeResults()` with conflict resolution.
- Update `updateGTFSRealtimePeriodically()` (minimal change - passes config through).
- Add startup agency ID validation warnings.
- Add merge unit tests.

### Phase 4: Integration Testing
- Create multi-feed integration test helper.
- Add integration tests for multi-agency vehicle queries.
- Add integration tests for cross-feed alert aggregation.
- Verify no regressions in existing single-feed tests.

---

## Open Questions

1. **Per-feed polling intervals**: Some agencies update their RT feeds more frequently than others. Should we support per-feed intervals in a future iteration?
2. **Feed health tracking**: Should individual feed failures be surfaced in the `/healthz` endpoint or a new status endpoint?
3. **ID namespacing**: If two agencies use the same trip or vehicle IDs (e.g., both use numeric IDs starting from 1), should we prefix IDs with the agency ID to avoid collisions? The current spec relies on config ordering for conflict resolution, which may not be sufficient for all deployments.
