***=== Install Node.JS===***

winget install OpenJS.NodeJS.LTS



node -v

npm -v

npx -v



***=== Install Git official ===***

winget install -e --id Git.Git



***=== Install Claude Code Via Npm ===***

npm install -g @anthropic-ai/claude-code



**=== Set Regedit untuk API dari Deepseek ===**

\[System.Environment]::SetEnvironmentVariable('ANTHROPIC\_API\_KEY', 'API\_KEY', 'User')

\[System.Environment]::SetEnvironmentVariable("ANTHROPIC\_BASE\_URL", "https://api.deepseek.com/anthropic", "User")

\[System.Environment]::SetEnvironmentVariable("ANTHROPIC\_MODEL", "deepseek-v4-flash", "User")

\[System.Environment]::SetEnvironmentVariable("ANTHROPIC\_DEFAULT\_OPUS\_MODEL", "deepseek-v4-pro\[1m]", "User")

\[System.Environment]::SetEnvironmentVariable("ANTHROPIC\_DEFAULT\_SONNET\_MODEL", "deepseek-v4-flash-vision-exp", "User")

\[System.Environment]::SetEnvironmentVariable("ANTHROPIC\_DEFAULT\_HAIKU\_MODEL", "deepseek-v4-flash", "User")

\[System.Environment]::SetEnvironmentVariable("CLAUDE\_CODE\_SUBAGENT\_MODEL", "deepseek-v4-flash", "User")

\[System.Environment]::SetEnvironmentVariable("CLAUDE\_CODE\_EFFORT\_LEVEL", "high", "User")

\[System.Environment]::SetEnvironmentVariable("CLAUDE\_CODE\_AUTO\_COMPACT\_WINDOW", "786432", "User")



\[System.Environment]::SetEnvironmentVariable("CLAUDE\_CODE\_ATTRIBUTION\_HEADER", "0", "User")

\[System.Environment]::SetEnvironmentVariable("CLAUDE\_CODE\_ENABLE\_TELEMETRY", "0", "User")

\[System.Environment]::SetEnvironmentVariable("CLAUDE\_CODE\_DISABLE\_NONESSENTIAL\_TRAFFIC", "1", "User")

\[System.Environment]::SetEnvironmentVariable("CLAUDE\_CODE\_SIMPLE", "1", "User")



**=== Setting.Json CLI ===**



{

&#x20; "permissions": {

&#x20;   "defaultMode": "bypassPermissions"

&#x20; },

&#x20; "model": "opus",

&#x20; "enabledPlugins": {

&#x20;   "frontend-design@claude-plugins-official": true,

&#x20;   "context7@claude-plugins-official": true,

&#x20;   "superpowers@claude-plugins-official": true

&#x20; },

&#x20; "skipDangerousModePermissionPrompt": true

}

