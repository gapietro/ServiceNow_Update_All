
# ServiceNow Update All - Runbook

**Current version:** 1.0.6 | **Scope:** `x_snc_update_all`

## Overview

ServiceNow Update All is a scoped application that automates the process of updating ServiceNow plugins and applications. It solves a common pain point for ServiceNow administrators who need to regularly update multiple applications across their instance.

Instead of manually updating each application one at a time through the UI, this application provides:
- Automated batch updating of plugins
- Scheduled weekly execution
- Configurable skip lists for sensitive applications
- Batch size control to prevent system overload
- Detailed logging

## Installation

There are two supported ways to install the application.

### Option 1: Update set

1. Import the update set (`ServiceNow_Update_All_1_0_6_UpdateSet.xml`) into your ServiceNow instance
2. Preview and commit the update set
3. The application will be installed with the scope `x_snc_update_all`

### Option 2: Source control

1. Create a **Basic Auth Credentials** record under Connections & Credentials > Credentials. Use your GitHub user name and a personal access token with the `repo` scope as the password. (A connection alias or HTTP connection is not used here; the Link dialog lists credential records only.)
2. In Studio, choose **Import From Source Control** and enter the repository URL, the credential and the branch. If the app is already installed, open it in Studio and choose **Source Control > Link to Source Control** instead.
3. When linking an existing app, note that the link **deletes the app's local update sets and customer update records**, and that it **commits the instance's current state to the chosen branch**. Link to a working branch, not `main`, and answer **No** to retaining update set history as commits.

## Required Permissions

The application runs in its own scope and needs these cross-scope privileges. They are shipped with the app:

| Target | Type | Operation |
|---|---|---|
| `AppUpgrader` | Script Include | Execute |
| `UpdateChecker` | Script Include | Execute |
| `sys_store_app` | Table | Read |
| `sys_progress_worker` | Table | Read |
| `Glide API: properties` | Scriptable | Execute |

The scheduled job runs as **System Administrator**.

## Configuration

The application uses the following system properties to control its behavior:

### x_snc_update_all.plugin.update.skip.list

* **Purpose**: Defines which plugins should be excluded from the automated update process
* **Type**: String (comma-separated list)
* **Default**: Empty (no skipped plugins)
* **Example**: `com.glide.service-mapping,com.glide.discovery,sys_id_of_plugin_to_skip`
* **Note**: You can use either the plugin `source` value or the `sys_id` of the `sys_store_app` record

### x_snc_update_all.plugin.update.batch.size

* **Purpose**: Controls how many plugins will be updated in a single execution
* **Type**: Integer
* **Default**: 500 (as shipped in the update set). If the property is missing or not a valid number, the script falls back to 10.
* **Recommendation**: Lower this value if you experience performance issues during updates. Only successful updates count toward the batch limit.

## Operation

### Scheduled Execution

By default, the application runs automatically as a scheduled job:

* **Job Name**: UpdateAll Plugins Weekly Job
* **Schedule**: Weekly on Sundays at 05:00 (job time zone: America/New_York). Confirm the exact time on the job record, since it is stored in the job's own time zone.
* **Table**: sysauto_script

You can modify the schedule through the Scheduled Jobs table in ServiceNow.

### Manual Execution

There are two ways to manually execute the update process:

#### Option 1: Execute the Scheduled Job directly

1. Navigate to System Definition > Scheduled Jobs
2. Search for "UpdateAll Plugins Weekly Job"
3. Open the job record
4. Click the "Execute Now" button at the top of the form

#### Option 2: Run via Background Script

1. Navigate to System Definition > Scripts - Background
2. Create a new background script
3. Enter the following code:
   ```javascript
   var result = x_snc_update_all.UpdateAllPlugins();
   gs.print(JSON.stringify(result, null, 2));
   ```
4. Click "Run script"

### Timing

A run is slow, so expect to wait. In a verification run on a development instance, refreshing the store data took about 3 minutes and upgrading a single plugin (Admin Center) took about 3 more. The job returns after each upgrade call completes, so a full batch can run for a long time. Start with a small batch size when testing.

## Logging

The application generates detailed logs that can be viewed in the System Logs:

1. Navigate to System Logs > System Log
2. Filter for Source containing "x_snc_update_all"

The logs include:
- Start and completion of the update process
- Store data refresh status
- Number of plugins found for updating
- Skipped plugins (if any)
- Status of each plugin update (queued, complete or failed)
- A summary line with updated, skipped and failed counts and the total duration

Example of a successful single-plugin run:

```
Plugin update process (x_snc_update_all) ▸ Found 33 updatable plugins
Plugin update (x_snc_update_all) ▸ QUEUED Admin Center [6.3.0 → 6.4.5]
Plugin update (x_snc_update_all) ▸ COMPLETE Admin Center (took 175.816s)
Plugin update process (x_snc_update_all) ▸ Processed batch of 1. Remaining: 32
Plugin update summary (x_snc_update_all) ▸ Updated: 1, Skipped: 0, Failed: 0, Duration: 342.431s
```

## Troubleshooting

### Common Issues

1. **Error: Failed to refresh store data**
   * Cause: Connection issues with the ServiceNow Store
   * Solution: Check network connectivity to the ServiceNow Store, verify proxy settings if applicable. The run continues with the data already on the instance.

2. **No plugins found for update**
   * Cause: The store data may not be refreshed or all plugins are already up to date
   * Solution: Wait for the store to refresh or manually check for updates in the Application Navigator

3. **Updates failing for specific plugins**
   * Cause: Plugin-specific issues or dependencies
   * Solution: Check the error message in the logs and try updating the plugin manually

4. **Access or privilege errors in the log** (for example mentioning `AppUpgrader`, `UpdateChecker`, `sys_store_app` or `sys_progress_worker`)
   * Cause: A required cross-scope privilege is missing or not allowed
   * Solution: Check the list under Required Permissions against the app's cross-scope privileges and set any missing one to Allowed

### Restarting Failed Updates

If updates fail for some plugins, you can retry by:
1. Adding the successfully updated plugins to the skip list 
2. Running the update process again

## Best Practices

1. **Schedule during off-hours**: Run updates during periods of low system usage
2. **Skip critical applications**: Add mission-critical applications to the skip list
3. **Test updates**: Consider testing updates in a sub-production environment first
4. **Monitor logs**: Regularly check the logs to ensure the update process is working correctly
5. **Adjust batch size**: Fine-tune the batch size based on your instance's performance
6. **Restore settings after testing**: If you lower the batch size to test, set it back afterwards

## Technical Details

The main script include, `UpdateAllPlugins`, performs these operations:
1. Refreshes the store data to get the latest plugin information (failures are logged and do not stop the run)
2. Queries the `sys_store_app` table for updatable plugins
3. Skips any plugin whose `source` or `sys_id` is in the skip list
4. Upgrades the remaining plugins one at a time with `AppUpgrader`, up to the batch size
5. Logs detailed information about the update process
6. Returns a summary: `{ updated: [], skipped: [], failed: [], duration: seconds }`

### Known limitations

* Errors from the setup code (creating the upgrader and querying `sys_store_app`) and from the scheduled job wrapper are not caught, so such a failure ends the run without a summary.
* A plugin is counted as updated as soon as the upgrade call returns without throwing. A failure reported later is not captured in the `failed` list.
* The "Remaining" figure logged after a full batch includes skipped plugins.

## Version History

| Version | Notes |
|---|---|
| 1.0.6 | Renamed `id` to `sysId` in `UpdateAllPlugins` and the source-control copy synced with the update set; added the `sys_progress_worker` privilege to source control |
| 1.0.5 | Previous release |
| 1.0.4 | Refresh the store data before checking for updates |

## Support

For issues with this application, please contact your ServiceNow administrator or the application developer. 
