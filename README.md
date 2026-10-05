# APULODI Agent Skills

Official [Agent Skills](https://skills.sh) for building integrations with [APULODI](https://apulodi.dev) — developer-first file infrastructure for uploading, storing, transforming and delivering files.

These skills give AI agents (Claude Code, Cursor, Codex, Amp, ...) verified, current knowledge of the APULODI API so they write correct integrations instead of hallucinating endpoints.

## Install

```bash
# Umbrella skill (overview + routing to sub-skills)
npx skills add hassiekond/apulodi-skills@apulodi

# Or install specific sub-skills
npx skills add hassiekond/apulodi-skills@apulodi-file-uploads
npx skills add hassiekond/apulodi-skills@apulodi-media-transforms
npx skills add hassiekond/apulodi-skills@apulodi-webhooks
npx skills add hassiekond/apulodi-skills@apulodi-management
```

## Skills

| Skill | What it teaches |
| --- | --- |
| `apulodi` | What APULODI is, auth model, base URL, response envelopes, SDK install, routing to sub-skills |
| `apulodi-file-uploads` | The 3-step direct-to-storage upload flow, multipart uploads for large files, listing, lifecycle ops (rename/copy/restore/delete), download URLs |
| `apulodi-media-transforms` | Image/video/audio transformations, the async pending→ready contract, signed variant URLs, transform-at-upload patterns |
| `apulodi-webhooks` | HMAC signature verification (TS + Python), replay protection, event dedup, the full event catalog, drop-in receivers |
| `apulodi-management` | Organizations, projects, team invitations and roles, API-key administration (session-authenticated dashboard APIs) |

## SDKs

The skills reference the official SDKs:

- TypeScript: `npm install @apulodi/sdk`
- Python: `pip install apulodi`

## Docs

Full API documentation: https://apulodi.dev/docs

## License

MIT
