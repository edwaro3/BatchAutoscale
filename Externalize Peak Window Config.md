# Proposed Change: Externalize Peak Window Configuration

This document describes a proposed change to move the hardcoded peak window schedule, preload percentage, and timezone out of the orchestrator code and into an externally managed configuration file. This allows the schedule to be changed at any time — including for a single day — without a code change or redeployment.

---

## Problem

The peak window times, preload percentage, and timezone are currently hardcoded in the orchestrator (`CloudPool pool Query Combined.cs`):

```csharp
bool isPeakWindow = (localTime >= new TimeOnly(7, 45) && localTime < new TimeOnly(10, 0))
                 || (localTime >= new TimeOnly(11, 30) && localTime < new TimeOnly(13, 0))
                 || (localTime >= new TimeOnly(15, 30) && localTime < new TimeOnly(17, 30));

int preloadMinJobs = isPeakWindow ? (int)Math.Ceiling(maxJobs * 0.30) : 0;
```

To change any of these values — even temporarily for a single day — requires a code change, a build, and a deployment. This is slow and disproportionate for what should be an operational tuning decision.

---

## Proposed solution

Store the peak window configuration in an **Azure Blob Storage JSON file** within the existing Storage Account (already used by Durable Functions and `_scaleLockStore`). The orchestrator reads this file instead of using hardcoded values.

### Why Blob Storage

| Option | Verdict |
|--------|---------|
| **Azure Blob Storage** | Reuses existing Storage Account. No new Azure resource. Supports structured JSON with date overrides. Editable via Portal, Storage Explorer, or CLI. |
| **Azure App Configuration** | Purpose-built for dynamic config. Has change history, validation, and native .NET `IOptionsSnapshot` binding. Best long-term option but adds a new Azure resource (~£1/month). Could be adopted later if needs grow. |
| **Azure Key Vault** | Already in the system (holds the formula template) but designed for secrets, not frequently-changing schedules. Awkward to represent structured data like multiple time windows. |
| **Azure Table Storage** | Available (same Storage Account) and structured, but less convenient to hand-edit than a JSON file. |

Blob Storage is recommended as the starting point because it requires zero new infrastructure and is simple to implement. If the team later wants richer features (change history, validation, push-based refresh), migrating to Azure App Configuration is straightforward.

---

## Configuration schema

A single JSON file stored in a dedicated blob container (e.g. `config/peak-windows.json`):

```json
{
  "defaultWindows": [
    { "start": "07:45", "end": "10:00" },
    { "start": "11:30", "end": "13:00" },
    { "start": "15:30", "end": "17:30" }
  ],
  "overrides": [
    {
      "date": "2026-04-30",
      "windows": [
        { "start": "09:00", "end": "10:00" }
      ]
    }
  ],
  "preloadPercentage": 0.30,
  "timezone": "Central European Standard Time"
}
```

### Field reference

| Field | Type | Description |
|-------|------|-------------|
| `defaultWindows` | Array of `{ start, end }` | Recurring daily peak windows in `HH:mm` format (local time). Used on any day without a date-specific override. |
| `overrides` | Array of `{ date, windows }` | Date-specific schedules. `date` is `YYYY-MM-DD`. When an override matches today's date, its `windows` array **replaces** `defaultWindows` entirely for that day. |
| `overrides[].windows` | Array of `{ start, end }` | Same format as `defaultWindows`. Can be empty (`[]`) to disable preloading for that date. |
| `preloadPercentage` | Decimal (0–1) | Fraction of `maxJobs` to use as the warm floor during peak windows. Currently `0.30` (30%). |
| `timezone` | String | Windows timezone ID for local time conversion. Currently `"Central European Standard Time"`. |

### Override behaviour

- If an override exists for today's date, its `windows` array is used **instead of** (not merged with) `defaultWindows`. This keeps the mental model simple: an override gives you full control of that day's schedule.
- To disable all preloading for a specific date, set `"windows": []` in the override.
- Overrides for past dates have no effect and can be cleaned up periodically.

### Examples

**Narrow tomorrow's preload to 9–10 AM only:**
```json
{
  "date": "2026-04-30",
  "windows": [{ "start": "09:00", "end": "10:00" }]
}
```

**Disable preloading entirely on a specific date (e.g. a public holiday):**
```json
{
  "date": "2026-05-01",
  "windows": []
}
```

**Increase the preload floor to 50% during a promotional period:**
Change `"preloadPercentage": 0.50` in the config. No override needed — this applies globally until changed back.

---

## Code changes

### New files

#### 1. `PeakWindowConfig.cs` — Configuration model

```csharp
public sealed class PeakWindowConfig
{
    public List<TimeWindow> DefaultWindows { get; set; } = new();
    public List<DateOverride> Overrides { get; set; } = new();
    public double PreloadPercentage { get; set; } = 0.30;
    public string Timezone { get; set; } = "Central European Standard Time";
}

public sealed class TimeWindow
{
    public string Start { get; set; } = string.Empty;
    public string End { get; set; } = string.Empty;

    public TimeOnly StartTime => TimeOnly.Parse(Start);
    public TimeOnly EndTime => TimeOnly.Parse(End);
}

public sealed class DateOverride
{
    public string Date { get; set; } = string.Empty;
    public List<TimeWindow> Windows { get; set; } = new();

    public DateOnly ParsedDate => DateOnly.Parse(Date);
}
```

#### 2. `PeakWindowConfigReader.cs` — Blob reader with short-lived cache

```csharp
public sealed class PeakWindowConfigReader
{
    private readonly BlobClient _blobClient;
    private PeakWindowConfig? _cached;
    private DateTimeOffset _cacheExpiry = DateTimeOffset.MinValue;
    private static readonly TimeSpan CacheTtl = TimeSpan.FromMinutes(1);

    // Hardcoded fallback matching the original code — used if the blob
    // is missing or cannot be parsed, so the system never breaks.
    private static readonly PeakWindowConfig FallbackConfig = new()
    {
        DefaultWindows = new List<TimeWindow>
        {
            new() { Start = "07:45", End = "10:00" },
            new() { Start = "11:30", End = "13:00" },
            new() { Start = "15:30", End = "17:30" },
        },
        PreloadPercentage = 0.30,
        Timezone = "Central European Standard Time",
    };

    public PeakWindowConfigReader(BlobClient blobClient)
    {
        _blobClient = blobClient;
    }

    public async Task<PeakWindowConfig> GetConfigAsync()
    {
        if (_cached is not null && DateTimeOffset.UtcNow < _cacheExpiry)
            return _cached;

        try
        {
            BlobDownloadResult result = await _blobClient.DownloadContentAsync();
            var config = JsonSerializer.Deserialize<PeakWindowConfig>(
                result.Content.ToString(),
                new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

            _cached = config ?? FallbackConfig;
        }
        catch
        {
            // Blob missing, network error, or malformed JSON — use fallback.
            _cached = FallbackConfig;
        }

        _cacheExpiry = DateTimeOffset.UtcNow + CacheTtl;
        return _cached;
    }

    /// <summary>
    /// Returns the effective windows for the given date: date-specific override
    /// if one exists, otherwise the default recurring windows.
    /// </summary>
    public static List<TimeWindow> GetEffectiveWindows(PeakWindowConfig config, DateOnly today)
    {
        var match = config.Overrides.FirstOrDefault(o => o.ParsedDate == today);
        return match?.Windows ?? config.DefaultWindows;
    }
}
```

### Modified file: `CloudPool pool Query Combined.cs`

The hardcoded timezone, peak window conditions, and `0.30` preload percentage are replaced with a config reader call. Everything else in the file (pool query, cooldown check, `EnableAutoScaleAsync` call, lock store write) remains unchanged.

**Before** (current code):
```csharp
    TimeZoneInfo polandTz = TimeZoneInfo.FindSystemTimeZoneById("Central European Standard Time");
    TimeOnly localTime = TimeOnly.FromTimeSpan(
        TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, polandTz).TimeOfDay);

    bool isPeakWindow = (localTime >= new TimeOnly(7, 45) && localTime < new TimeOnly(10, 0))
                     || (localTime >= new TimeOnly(11, 30) && localTime < new TimeOnly(13, 0))
                     || (localTime >= new TimeOnly(15, 30) && localTime < new TimeOnly(17, 30));

    int preloadMinJobs = isPeakWindow ? (int)Math.Ceiling(maxJobs * 0.30) : 0;

    string formattedFormula = string.Format(this._scaleFormula, maxJobs, preloadMinJobs);
```

**After** (proposed):
```csharp
    // Read peak window config from blob storage (cached for ~1 min).
    PeakWindowConfig config = await this._peakWindowConfigReader.GetConfigAsync();

    TimeZoneInfo tz = TimeZoneInfo.FindSystemTimeZoneById(config.Timezone);
    TimeOnly localTime = TimeOnly.FromTimeSpan(
        TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, tz).TimeOfDay);

    DateOnly today = DateOnly.FromDateTime(TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, tz));
    List<TimeWindow> windows = PeakWindowConfigReader.GetEffectiveWindows(config, today);

    bool isPeakWindow = windows.Any(w =>
        localTime >= w.StartTime && localTime < w.EndTime);

    int preloadMinJobs = isPeakWindow
        ? (int)Math.Ceiling(maxJobs * config.PreloadPercentage)
        : 0;

    string formattedFormula = string.Format(this._scaleFormula, maxJobs, preloadMinJobs);
```

### App setting

Add a new app setting for the blob path so it isn't hardcoded:

| Setting | Example value |
|---------|---------------|
| `PeakWindowConfigBlobUri` | `https://<storageaccount>.blob.core.windows.net/config/peak-windows.json` |

The `BlobClient` is constructed from this URI and injected into `PeakWindowConfigReader` via DI.

---

## Deployment steps

1. **Upload the initial config blob** to the Storage Account with the current default values (matching the existing hardcoded windows). This ensures behaviour is identical before and after deployment.
2. **Deploy the code changes** (new files + refactored orchestrator).
3. **Add the `PeakWindowConfigBlobUri` app setting** to the Function App configuration.
4. **Verify** the orchestrator reads the config correctly by checking Application Insights logs.
5. **Enable blob versioning** (optional but recommended) on the storage container for change history / rollback.

---

## Graceful fallback

If the blob is missing, inaccessible, or contains invalid JSON, the `PeakWindowConfigReader` falls back to a hardcoded default matching the current production values. This means:

- The system **never breaks** due to a missing or corrupt config file.
- A bad edit to the JSON file does not cause an outage — it just reverts to the original behaviour until the file is fixed.
- Logs should record when the fallback is used so it doesn't go unnoticed.

---

## Operational workflow

### Changing tomorrow's schedule

1. Open the Storage Account in the Azure Portal → Containers → `config` → `peak-windows.json`.
2. Click **Edit**, add an entry to the `overrides` array:
   ```json
   { "date": "2026-04-30", "windows": [{ "start": "09:00", "end": "10:00" }] }
   ```
3. Click **Save**.
4. The orchestrator picks up the change on its next run (within 1–5 minutes).
5. On April 30, only the 09:00–10:00 window will trigger preloading. On May 1, the system automatically falls back to `defaultWindows`.

### Changing the default schedule permanently

1. Edit the `defaultWindows` array in the blob.
2. Save. Takes effect within 1–5 minutes.

### Changing the preload percentage

1. Edit `preloadPercentage` in the blob (e.g. `0.50` for 50%).
2. Save. Takes effect within 1–5 minutes.

---

## Verification plan

| # | Test | Type |
|---|------|------|
| 1 | `PeakWindowConfigReader` correctly parses sample JSON with default windows and no overrides | Unit |
| 2 | Date override for today replaces default windows | Unit |
| 3 | Date override for a different date is ignored; defaults used | Unit |
| 4 | Empty `overrides` array falls back to `defaultWindows` | Unit |
| 5 | Missing or malformed blob triggers fallback config | Unit |
| 6 | `preloadPercentage` from config is used instead of hardcoded `0.30` | Unit |
| 7 | Changing the blob content causes the orchestrator to pick up new windows after cache expiry | Integration |
| 8 | Upload a config with a narrow test window; confirm preload floor is applied only during that window | Manual |

---

## Future considerations

- **Azure App Configuration**: If the team later wants richer features (change history, validation, push-based refresh, feature flags), migrating from a blob to App Configuration is straightforward — the config model stays the same, only the reader changes.
- **Validation endpoint**: If non-developers need to change windows, a small HTTP-triggered Azure Function could wrap blob writes with JSON schema validation to prevent bad edits.
- **Cleanup automation**: A lightweight timer function could periodically remove overrides for past dates to keep the config file tidy.
- **Per-pool config**: If different pools need different schedules in the future, the blob path could include the pool ID (e.g. `config/peak-windows-{poolId}.json`).
