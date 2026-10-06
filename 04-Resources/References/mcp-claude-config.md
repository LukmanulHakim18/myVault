{

  "mcpServers": {

    "GitLab communication server": {

      "command": "npx",

      "args": ["-y", "@zereight/mcp-gitlab"],

      "env": {

        "GITLAB_PERSONAL_ACCESS_TOKEN": "YOUR_GITLAB_TOKEN_HERE",

        "GITLAB_API_URL": "https://git.bluebird.id/api/v4",

        "GITLAB_READ_ONLY_MODE": "false",

        "USE_GITLAB_WIKI": "false",

        "USE_MILESTONE": "false",

        "USE_PIPELINE": "false"

      }

    },

    "github": {

      "command": "npx",

      "args": ["-y", "@modelcontextprotocol/server-github"],

      "env": {

        "GITHUB_PERSONAL_ACCESS_TOKEN": "YOUR_GITHUB_TOKEN_HERE"

      }

    },

    "sequential-thinking": {

      "command": "npx",

      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]

    },

    "desktop-commander": {

      "command": "npx",

      "args": ["-y", "@wonderwhy-er/desktop-commander"]

    },

    "context7": {

      "command": "npx",

      "args": ["-y", "@upstash/context7-mcp"]

    },

    "obsidian": {

      "command": "npx",

      "args": ["-y", "mcp-obsidian", "D:\\myVault"]

    }

  },

  "preferences": {

    "chillingSlothLocation": {

      "customPath": "D:\\ClaudeAgents"

    },

    "coworkScheduledTasksEnabled": false,

    "sidebarMode": "chat"

  }

}