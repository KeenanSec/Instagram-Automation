# Instagram Automation via n8n

A lightweight, stateless n8n workflow that turns a Google Sheet into a scheduled Instagram content queue. 

## Architecture

![Instagram Automation Workflow](/workflow.png)

## How it works

The workflow runs on a cron schedule (e.g., every hour):
1. **Fetch:** Pulls rows from Google Sheets where `Status = 'Pending'` and the release time has passed.
2. **Stage:** Sends the `image_url` and `caption` to the Meta Graph API to create a media container.
3. **Publish:** Takes the resulting `creation_id` and publishes it live to Instagram.
4. **Log:** Updates the original spreadsheet row to `Status = 'Posted'`.

## Sheet Schema

Your Google Sheet needs these exact headers:

| Column | Description |
|---|---|
| `row_number` | Unique ID (used to match and update rows) |
| `File name` | Reference name for the asset |
| `Caption` | The Instagram post text |
| `Status` | Track state (`Pending`, `Posted`) |

## Setup

1. **Import:** Import `workflow.json` into your n8n instance.
2. **Google Sheets:** Connect your Google Service Account. Update the **Get** and **Update** nodes with your specific Spreadsheet ID and Sheet Name.
3. **Instagram/Meta API:** Configure your Facebook Graph API credentials. Ensure your Meta system user token has `instagram_content_publish` permissions and a linked IG Business/Creator account.
4. **Test:** Add one row to your sheet, trigger the workflow manually, and verify the post goes live.

## Known Limitations

* **Single Images Only:** Does not support Carousels or Reels yet.
* **No Error Catching:** If the Meta API fails, the sheet doesn't automatically update to "Failed." You'll need to add an error trigger branch if you want error logging.
* **Token Expiration:** Meta tokens expire every 60 days.
