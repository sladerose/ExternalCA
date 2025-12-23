# Supabase RPC and View Documentation

This document outlines the Supabase database objects (Remote Procedure Calls and Views) integrated into the `sites_down` application. These objects provide the data foundation for the dashboard visualizations.

## 1. Sites Down (RPC)
- **Function Name**: `get_sites_down_last_6_hours_with_downtime`
- **Purpose**: Identifies sites that have been inactive (disconnected) for more than 6 hours.
- **Key Metrics**:
    - `downtime_duration`: Human-readable duration string (e.g., "1 day 2 hours").
    - `last_file_activity_time`: Timestamp of the last successful file transfer.
- **Used In**: "Sites Currently Down" Monitor Card.

## 2. File Pulse (View)
- **View Name**: `view_site_file_pulse`
- **Purpose**: Tracks the "freshness" of different file types for each station to identify specific data pipeline delays.
- **Key Fields**:
    - `last_pos`: Timestamp of last Point of Sale file.
    - `last_bors`: Timestamp of last Back Office Report file.
    - `last_delivs`: Timestamp of last Delivery file.
    - `last_site_sta`: Timestamp of last Station Status file.
- **Used In**: "File Freshness" Tab.

## 3. Regional Health (View)
- **View Name**: `view_regional_health`
- **Purpose**: Aggregates site connectivity metrics by Province and Zone to provide a high-level health overview.
- **Key Metrics**:
    - `total_sites`: Total count of sites in the region.
    - `sites_down`: Count of sites currently down in the region.
    - `uptime_percentage`: Percentage of sites currently online.
- **Used In**: "Regional Health" Tab (Heatmap and Table).

## 4. Site Reliability Report (View)
- **View Name**: `view_site_reliability_report`
- **Purpose**: Analyzes historical stability over the last 7 days to identify "flapping" or unstable sites.
- **Key Metrics**:
    - `incidents_last_7_days`: Number of times connectivity dropped for > 1 hour.
    - `reliability_status`: 'Stable', 'Unstable', or 'Critical - Flapping'.
- **Used In**: "Site Monitoring" Tab (Reliability Table).

## 5. Duplicate File Audit (View)
- **View Name**: `view_duplicate_file_audit`
- **Purpose**: Technical audit to detect files that have been processed multiple times, indicating potential pipeline inefficiencies.
- **Key Fields**:
    - `processing_count`: Number of times the file was seen.
    - `first_seen` / `last_seen`: Timestamps of first and most recent processing.
- **Used In**: "Technical Audit" Tab.

## 6. Sequence Gaps (RPC)
- **Function Name**: `get_site_sequence_gaps`
- **Purpose**: Identifies breaks in the sequence numbers of files for a specific station, useful for deep-dive debugging of data loss.
- **Parameters**: `station_id`, `file_type`.
- **Returns**: Gaps in sequence numbers.
- **Used In**: Backend service logic (available for future detailed drill-downs).

---

## Appendix: SQL Definitions

### get_sites_down_last_6_hours_with_downtime
```sql
-- (See source code or database for full definition)
-- Calculates diff between NOW() and last_file_activity_time
```

### view_regional_health
```sql
CREATE OR REPLACE VIEW public.view_regional_health AS
SELECT 
    sm.province,
    sm.zone,
    COUNT(sm.station_id) as total_sites,
    COUNT(sm.station_id) FILTER (WHERE sm.last_event_time < NOW() - INTERVAL '6 hours') as sites_down,
    ROUND((COUNT(sm.station_id) FILTER (WHERE sm.last_event_time >= NOW() - INTERVAL '6 hours')::numeric / 
    NULLIF(COUNT(sm.station_id), 0)) * 100, 2) as uptime_percentage
FROM public.astron_site_master sm
GROUP BY sm.province, sm.zone
ORDER BY uptime_percentage ASC;
```

### view_site_file_pulse
```sql
CREATE OR REPLACE VIEW public.view_site_file_pulse AS
SELECT 
    sm.station_id,
    sm.station_name,
    sm.province,
    MAX(CASE WHEN fal.file_type = 'pos' THEN fal.event_time END) as last_pos,
    MAX(CASE WHEN fal.file_type = 'bors' THEN fal.event_time END) as last_bors,
    MAX(CASE WHEN fal.file_type = 'delivs' THEN fal.event_time END) as last_delivs,
    MAX(CASE WHEN fal.file_type = 'site_sta' THEN fal.event_time END) as last_site_sta
FROM public.astron_site_master sm
LEFT JOIN public.astron_file_activity_log fal ON sm.station_id = fal.station_id_station_id
WHERE sm.connected = TRUE
GROUP BY sm.station_id, sm.station_name, sm.province;
```
