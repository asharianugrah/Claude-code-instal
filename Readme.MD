=== Install Node.JS===
winget install OpenJS.NodeJS.LTS

node -v
npm -v
npx -v

=== Install Git official ===
winget install -e --id Git.Git

=== Install Claude Code Via Npm ===
npm install -g @anthropic-ai/claude-code

=== Set Regedit untuk API dari Deepseek ===
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_API_KEY', 'API_KEY', 'User')
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.deepseek.com/anthropic", "User")
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "deepseek-v4-flash", "User")
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_OPUS_MODEL", "deepseek-v4-pro[1m]", "User")
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_SONNET_MODEL", "deepseek-v4-flash-vision-exp", "User")
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", "deepseek-v4-flash", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_SUBAGENT_MODEL", "deepseek-v4-flash", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_EFFORT_LEVEL", "high", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_AUTO_COMPACT_WINDOW", "786432", "User")

[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_ATTRIBUTION_HEADER", "0", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_ENABLE_TELEMETRY", "0", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC", "1", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_SIMPLE", "1", "User")

=== Setting.Json CLI ===

{
  "permissions": {
    "defaultMode": "bypassPermissions"
  },
  "model": "opus",
  "enabledPlugins": {
    "frontend-design@claude-plugins-official": true,
    "context7@claude-plugins-official": true,
    "superpowers@claude-plugins-official": true
  },
  "skipDangerousModePermissionPrompt": true
}