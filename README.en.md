# AgilFrame Edge

[Русский](README.md) · **English**

AgilFrame Edge publishes a prepared MP4 file from an embedded Linux device into an AgilFrame SFU room. The Node.js agent sends video and audio through FFmpeg to server-side LiveKit Ingress over RTMP or RTMPS without transcoding. The source plays in a loop.

[AgilFrame project](https://github.com/GillHD/agilframe) · [Core](https://github.com/GillHD/agilframe-core)

## Prerequisites

- Install Node.js 20 or later, FFmpeg, and FFprobe on the device.
- Prepare an MP4 with one H.264 video track (`yuv420p`) and one AAC-LC audio track. The audio track can have one or two channels at 44.1 or 48 kHz.
- Enable Redis for Flows in Core, configure managed RTMP input in LiveKit Ingress, and create an SFU room in the tenant that owns the API key. [Core and LiveKit Ingress setup](https://github.com/GillHD/agilframe-core/blob/main/agil/core/docs/flows.md#managed-rtmp-push)

## Publish to a room

Run the agent from the repository root under a dedicated account on a trusted device. Pass service origins without paths or keys to `--core` and `--ingress`. Pass an absolute file path to `--input`. HTTP and RTMP are allowed only for local checks on a numeric loopback address. For RTMPS, the Node.js agent verifies the certificate and server name before sending media. FFmpeg connects through a temporary local port.

```bash
read -rs -p 'Tenant API key: ' AGIL_EDGE_CORE_API_KEY
export AGIL_EDGE_CORE_API_KEY
node agent/run-rtmp-publisher.mjs \
  --core https://core.example.org \
  --ingress rtmps://publish.example.org:443 \
  --room studio \
  --input /srv/edge/media/studio.mp4
unset AGIL_EDGE_CORE_API_KEY
```

The agent creates a separate Flow for each run and logs its ID and status. A `ready` status means Ingress has confirmed receipt of the stream. If Ingress temporarily stops confirming receipt, the agent switches to `degraded`. After 45 seconds without confirmation, the agent stops publishing and removes its Flow.

On `SIGINT` or `SIGTERM`, the agent stops FFmpeg and requests deletion of its Flow. Core revokes the input when it deletes the Flow. If the Core response is lost, use the logged Flow ID to locate it and remove it through the dashboard or API if necessary.

The agent keeps the Core API key. Local processes that can inspect FFmpeg arguments may see the publishing key.

## Run tests

With FFmpeg and FFprobe installed, run `npm test` from the repository root. The tests run without installing npm packages.

To include the media transport tests, provide the path to MediaMTX in the test environment. MediaMTX acts as the server-side receiver in these tests. In a deployment, the device publishes to LiveKit Ingress.

```bash
AGIL_QA_MEDIAMTX_BIN=/absolute/path/to/mediamtx npm test
```

The tests check H.264 and AAC reception, stream shutdown, and RTMPS certificate name validation. The separate Core end-to-end scenario checks decoded video and audio in a browser and confirms that the input is revoked after shutdown. [End-to-end test instructions](https://github.com/GillHD/agilframe-core/tree/main/agil/edge)

[License](LICENSE) · [Notices](NOTICE) · [Contributing](CONTRIBUTING.en.md) · [Security](SECURITY.en.md)
