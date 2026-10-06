# Security boundaries

Status: design requirements, not verified controls.

## Proposed first version

No private keys, seed phrases, exchange write credentials, transaction signing, automatic buying, or automatic selling. The meaning of an actual wallet remains a separate scope decision.

## Authorization

- Only an explicitly configured and paired device ID may read personal holdings or create a journal draft.
- Only an authenticated reply from that device and the current connection session may confirm a draft.
- Drafts expire after 60 seconds. Replayed, foreign, expired, disconnected, or superseded prompts never write.
- The agent has read/view/draft tools, but no direct journal-commit tool.
- Verify the effective tool catalog. A prompt instruction is not a sandbox.
- General terminal, file, code-execution, delegation, and unrelated MCP capabilities must be absent from the portfolio-device agent.
- A firmware-generated confirmation is not hardware-wallet-grade proof of human presence.

## Transport and voice

Use certificate-verified WSS before processing real holdings or voice. No certificate-validation bypass and no automatic public tunnel.

Use push-to-talk. Do not enable the camera or an always-on microphone. Confirm speech/model providers, costs, logging, retention, and data processing before live use.

## Local data

Use a separate Hermes runtime, private directory permissions, and an appropriate operator backup/encryption policy. Never copy another profile's credentials or personal memories.

The Git ignore rules cover common secret, runtime, database, and recording paths. Ignore rules do not replace reviewing the exact staged diff before every push.

## Publication boundary

This repository initially publishes documentation and folder scaffolding only. It contains no personal holdings, live market snapshot, audio, firmware image, or credential file.

Publishing source later requires separate review of the exact diff. Changing this project's public status does not authorize publishing any unrelated business artifacts or Hermes configuration.
