# ServiceNow Update All - Runbook

## Overview

ServiceNow Update All is a scoped application that automates the process of updating ServiceNow plugins and applications. It solves a common pain point for ServiceNow administrators who need to regularly update multiple applications across their instance.

Instead of manually updating each application one at a time through the UI, this application provides:
- Automated batch updating of plugins
- Scheduled weekly execution
- Configurable skip lists for sensitive applications
- Batch size control to prevent system overload
- Detailed logging

## Installation

1. Import the update set into your ServiceNow instance
2. Commit the update set
3. The application will be installed with the scope `x_snc_update_all`

## Configuration

The application uses the following system properties to control its behavior:

### x_snc_update_all.plugin.update.skip.list

* **Purpose**: Defines which plugins should be excluded from the automated update process
* **Type**: String (comma-separated list)
* **Default**: Empty (no skipped plugins)
* **Example**: `com.glide.service-mapping,com.glide.discovery,sys_id_of_plugin_to_skip`
* **Note**: You can use either the plugin source identifier or the sys_id

### x_snc_update_all.plugin.update.batch.size

* **Purpose**: Controls how many plugins will be updated in a single execution
* **Type**: Integer
* **Default**: 600
* **Recommendation**: Lower this value if you experience performance issues during updates

## Operation

### Scheduled Execution

By default, the application runs automatically as a scheduled job:

* **Job Name**: UpdateAll Plugins Weekly Job
* **Schedule**: Weekly on Sundays at 12:00 AM
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

## Logging

The application generates detailed logs that can be viewed in the System Logs:

1. Navigate to System Logs > System Log
2. Filter for Source containing "x_snc_update_all"

The logs include:
- Start and completion of the update process
- Number of plugins found for updating
- Skipped plugins (if any)
- Status of each plugin update (success or failure)
- Total duration of the update process

## Troubleshooting

### Common Issues

1. **Error: Failed to refresh store data**
   * Cause: Connection issues with the ServiceNow Store
   * Solution: Check network connectivity to the ServiceNow Store, verify proxy settings if applicable

2. **No plugins found for update**
   * Cause: The store data may not be refreshed or all plugins are already up to date
   * Solution: Wait for the store to refresh or manually check for updates in the Application Navigator

3. **Updates failing for specific plugins**
   * Cause: Plugin-specific issues or dependencies
   * Solution: Check the error message in the logs and try updating the plugin manually

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

## Technical Details

The main script include, `UpdateAllPlugins`, performs these operations:
1. Refreshes the store data to get the latest plugin information
2. Queries the `sys_store_app` table for updatable plugins
3. Processes updates in batches as defined by the batch size property
4. Logs detailed information about the update process
5. Returns a summary of the update operation

## Support

For issues with this application, please contact your ServiceNow administrator or the application developer. 
