# How to Ingest YouTube Playlist Data to Port and Create Entities Using GitHub Workflows

This guide demonstrates how to ingest information from a YouTube playlist into Port and create entities from each playlist video using GitHub workflows.

## Prerequisites
Before we start, you must connect a data source. For this task, we’ll use GitHub.

### Install Port’s GitHub App
Follow these steps to install Port’s GitHub app:

1. Go to the GitHub App page.
2. Click on the **Configure** button.
3. Choose the organization in which to install the app.
4. Within the selected organization, choose the repositories in which to install the app.
5. Click on the **Install** button.
6. Once the installation is complete, you will be redirected to Port.

### Add GitHub as a Data Source

1. Go to your **Data sources** in Port.
2. Click on **+ Data source**.
3. Select **GitHub cloud**.

## Create a GitHub Workflow

Create a new repository with a manual workflow. While there are multiple ways to POST data to Port, this guide uses a GET API call to fetch playlist items and each video’s information.

### GitHub Workflow YAML File
Paste the following code into your `.yml` file:

```yaml
name: Ingest YouTube Playlist Data

on:
  workflow_dispatch:
    inputs:
      port_context:
        description: "The port context input"
        required: false
        default: ""

jobs:
  fetch-playlist:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install requests

      - name: Fetch playlist data from YouTube API
        env:
          YOUTUBE_API_KEY: ${{ secrets.YOUTUBE_API_KEY }}
        run: |
          echo 'import requests
          import json
          import os

          playlist_id = "PL5ErBr2d3QJH0kbwTQ7HSuzvBb4zIWzhy"
          api_url = "https://www.googleapis.com/youtube/v3/playlistItems"

          all_items = []
          next_page_token = None

          while True:
              params = {
                  "part": "snippet",
                  "playlistId": playlist_id,
                  "maxResults": 50,
                  "key": os.getenv("YOUTUBE_API_KEY"),
              }
              if next_page_token:
                  params["pageToken"] = next_page_token

              response = requests.get(api_url, params=params)

              if response.status_code == 200:
                  data = response.json()

                  for item in data.get("items", []):
                      snippet = item.get("snippet", {})
                      all_items.append({
                          "videoId": snippet.get("resourceId", {}).get("videoId"),
                          "title": snippet.get("title"),
                          "description": snippet.get("description"),
                          "thumbnail": snippet.get("thumbnails", {}).get("default", {}).get("url"),
                          "publishedAt": snippet.get("publishedAt"),
                      })

                  next_page_token = data.get("nextPageToken")
                  if not next_page_token:
                      break
              else:
                  print(f"Error fetching playlist: {response.status_code} {response.text}")
                  exit(1)

          with open("playlist_items.json", "w") as f:
              json.dump(all_items, f, indent=2)

          print(f"Fetched {len(all_items)} items from the playlist.")

          os.makedirs("artifacts", exist_ok=True)
          os.rename("playlist_items.json", "artifacts/playlist_items.json")
          ' > fetch_playlist.py

          python fetch_playlist.py

      - name: Upload playlist data as artifact
        uses: actions/upload-artifact@v3
        with:
          name: playlist-items
          path: artifacts/playlist_items.json

  ingest-playlist:
    runs-on: ubuntu-latest
    needs: fetch-playlist

    steps:
      - name: Download playlist items artifact
        uses: actions/download-artifact@v3
        with:
          name: playlist-items
          path: .

      - name: Debug and Ingest playlist items into Port
        run: |
          echo "Reading playlist items from file..."
          PLAYLIST_ITEMS=$(cat playlist_items.json)
          echo "Playlist Items (full data):"
          echo "$PLAYLIST_ITEMS"

          if [[ $(echo "$PLAYLIST_ITEMS" | jq 'type') == '"array"' ]]; then
            echo "Valid JSON array detected."

            echo "$PLAYLIST_ITEMS" | jq -c '.[]' | while IFS= read -r ITEM; do
              VIDEO_ID=$(echo "$ITEM" | jq -r '.videoId')
              TITLE=$(echo "$ITEM" | jq -r '.title')
              DESCRIPTION=$(echo "$ITEM" | jq -r '.description')
              THUMBNAIL=$(echo "$ITEM" | jq -r '.thumbnail')
              PUBLISHED_AT=$(echo "$ITEM" | jq -r '.publishedAt')

              PAYLOAD=$(jq -n \
                --arg identifier "$VIDEO_ID" \
                --arg title "$TITLE" \
                --arg icon "Package" \
                --arg videoId "$VIDEO_ID" \
                --arg description "$DESCRIPTION" \
                --arg thumbnail "$THUMBNAIL" \
                --arg publishedAt "$PUBLISHED_AT" \
                '{
                  identifier: $identifier,
                  title: $title,
                  icon: $icon,
                  properties: {
                    videoId: $videoId,
                    title: $title,
                    description: $description,
                    thumbnail: $thumbnail,
                    publishedAt: $publishedAt
                  },
                  relations: {}
                }')

              echo "Payload to be ingested:"
              echo "$PAYLOAD"

              curl -L 'https://api.getport.io/v1/blueprints/dependency/entities' \
                -H 'Content-Type: application/json' \
                -H "Authorization: ${{ secrets.PORT_API_TOKEN }}" \
                -d "$PAYLOAD"
            done
          else
            echo "Error: Invalid JSON array detected."
            exit 1
          fi
```

### Add Secrets

1. Add `PORT_API_TOKEN` and `YOUTUBE_API_KEY` to the repository secrets.
2. To generate `PORT_API_TOKEN`, go to **Credentials**, click **Generate API token**, and copy the token into the GitHub secrets.
3. To generate your `YOUTUBE_API_KEY`, follow the [YouTube API guide](https://developers.google.com/youtube/registering_an_application).

## Add a Blueprint

Go to Port’s **Builder**, open the **Data model** tab, and click **+ Blueprint**. Then, click **Edit JSON**, paste the following code, and click **Save**:

```json
{
  "identifier": "dependency",
  "title": "Dependency",
  "icon": "Package",
  "schema": {
    "properties": {
      "videoId": {
        "icon": "DefaultProperty",
        "type": "string",
        "title": "Video ID"
      },
      "title": {
        "type": "string",
        "title": "Video title"
      },
      "description": {
        "type": "string",
        "title": "Description"
      },
      "thumbnail": {
        "type": "string",
        "title": "Thumbnail URL",
        "format": "url"
      },
      "publishedAt": {
        "type": "string",
        "title": "Publish date"
      },
      "viewCount": {
        "type": "number",
        "title": "View Count"
      },
      "likeCount": {
        "type": "number",
        "title": "Like Count"
      },
      "dislikeCount": {
        "type": "string",
        "title": "Dislike Count"
      },
      "commentCount": {
        "type": "number",
        "title": "Comment Count"
      },
      "durationInSeconds": {
        "type": "number",
        "title": "Duration in Seconds"
      }
    },
    "required": [
      "videoId",
      "title",
      "description",
      "thumbnail",
      "publishedAt"
    ]
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "aggregationProperties": {},
  "relations": {}
}
```

This will create a Blueprint code to generate an entity with the values sent by the GitHub workflow.

## Add an Action

Go to **Self-Service**, click **+ Action**, click **Edit JSON**, and paste the following code:

```json
{
  "identifier": "ingest",
  "title": "ingest",
  "icon": "Airflow",
  "trigger": {
    "type": "self-service",
    "operation": "CREATE",
    "userInputs": {
      "properties": {},
      "required": [],
      "order": []
    },
    "blueprintIdentifier": "dependency"
  },
  "invocationMethod": {
    "type": "GITHUB",
    "org": "MightyFrost",
    "repo": "port_yt",
    "workflow": "workflow.yml",
    "workflowInputs": {
      "{{ spreadValue() }}": "{{ .inputs }}",
      "port_context": {
        "runId": "{{ .run.id }}",
        "blueprint": "{{ .action.blueprint }}"
      }
    },
    "reportWorkflowStatus": true
  },
  "requiredApproval": false
}
```

Populate the `org`, `repo`, and `workflow` fields with your own information.

## Configure Exporter Mapping

Go to your **Data sources**, click **GitHub** under **Exporters**, and paste the following code into the **Mapping** window:

```yaml
resources:
  - kind: file
    selector:
      query: 'true'
    port:
      itemsToParse: .file.content.items | to_entries
      entity:
        mappings:
          identifier: >-
            .item.value.videoId + "_" + (.item.value.title | gsub("\\^";
            "caret_") | gsub("~"; "tilde_") | gsub(">="; "gte_") |  gsub("<=";
            "lte_") | gsub(">"; "gt_") |  gsub("<"; "lt_") |  gsub("@"; "at_") |
            gsub("\\*"; "star") |  gsub(" "; "_"))
          title: .item.value.title
          blueprint: '"dependency"'
          properties:
            videoId: .item.value.videoId
            title: .item.value.title
            description: .item.value.description
            thumbnail: .item.value.thumbnail
            publishedAt: .item.value.publishedAt
            viewCount: .item.value.viewCount
            likeCount: .item.value.likeCount
            dislikeCount: .item.value.dislikeCount
            commentCount: .item.value.commentCount
            durationInSeconds: .item.value.durationInSeconds
```

This ensures the data sent by the workflow is mapped correctly to the Blueprint.

## Create a Catalog Page

Go to the **Catalog** page, click **+ New**, and select **New catalog page**. Pick a title, select the created Blueprint (`Dependency`), and click **Create**.

Click **+ Dependency**, then **Ingest**, and **Execute**. This runs your GitHub workflow and starts populating the catalog with playlist items. Click on each catalog item to see the values returned by the YouTube API.

## Add Scorecards

Go to **Build**, find the `Dependency` Blueprint, click the **Scorecards** tab, and click **+ New scorecard**. Paste the following code:

```json
{
  "identifier": "dependency",
  "title": "Dependency",
  "levels": [
    {
      "color": "paleBlue",
      "title": "Below 250k views"
    },
    {
      "color": "bronze",
      "title": "Below 500k views"
    },
    {
      "color": "silver",
      "title": "Below 750k views"
    },
    {
      "color": "gold",
      "title": "Above 750k views"
    }
  ],
  "rules": [
    {
      "identifier": "viewCountBronze",
      "title": "View Count >= 500000",
      "level": "Below 500k views",
      "query": {
        "combinator": "and",
        "conditions": [
          {
            "operator": ">=",
            "property": "viewCount",
            "value": 500000
          }
        ]
      }
    },
    {
      "identifier": "viewCountSilver",
      "title": "View Count >= 750000",
      "level": "Below 750k views",
      "query": {
        "combinator": "and",
        "conditions": [
          {
            "operator": ">=",
            "property": "viewCount",
            "value": 750000
          }
        ]
      }
    },
    {
      "identifier": "viewCountGold",
      "title": "View Count >= 750000",
      "level": "Above 750k views",
      "query": {
        "combinator": "and",
        "conditions": [
          {
            "operator": ">=",
            "property": "viewCount",
            "value": 750000
          }
        ]
      }
    }
  ]
}
```

This evaluates the `viewCount` (number of views) and assigns the videos into the correct range.

## Create a Scorecard Dashboard

Go to your **Catalog**, click **+ New**, and select **New scorecard dashboard**. Choose a title, and select `Dependency` under Blueprint and Scorecard.

Port automatically generates data visualizations, but you can add your own as well. For example, add a pie chart that breaks videos down by length:

1. Click **+ Widget**.
2. Select **Pie chart**.
3. Choose a title and select the `Dependency` Blueprint, with **Duration in Seconds** for Breakdown by property.

These examples demonstrate how to visualize data in Port to evaluate the performance of your YouTube videos. You can compare views to comments, analyze engagement, and refine your content strategy.
