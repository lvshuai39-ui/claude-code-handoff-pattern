## What I just did
- Migrated data pipeline from cloud function to dedicated VM
- Deployed collector + dashboard as systemd services
- Set up 3 cron jobs: morning run / evening run / health check
- Configured webhook notifications for failures

## Current status
Pipeline is live on VM. End-to-end verified (collect → process → notify). Cloud function decommissioned.

## Next steps
- Watch first automated run tomorrow morning
- API token expires in ~30 days — alert will fire when it does
- VM auto-restart tested but uptime TBD under load
