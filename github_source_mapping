resources:
  - kind: file
    selector:
      query: 'true'
      files:
        - path: '**/playlist_data.json'
          repos:
            - port_yt
    port:
      itemsToParse: .file.content.items | to_entries
      entity:
        mappings:
          identifier: >-
            .item.value.videoId + "_" + (.item.value.title | gsub("\\^";
            "caret_") | gsub("~"; "tilde_") |  gsub(">="; "gte_") |  gsub("<=";
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
