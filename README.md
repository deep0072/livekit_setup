# livekit_setup


# LiveKit Server Setup with Recording (Egress)

This guide details the steps to set up a LiveKit server, configure the Egress service for recording sessions, and initiate a recording.

## Prerequisites

*   A Linux or macOS machine (commands are Unix-based).
*   **Docker:** Required to run the Egress service. Install Docker Engine if you haven't already.
*   **Redis:** A Redis instance must be running and accessible from both the machine running the LiveKit server and the machine running the Egress service (or the Docker host).
    *   Ensure Redis is not bound only to `127.0.0.1` if services are on different machines or in Docker.
    *   Ensure any firewalls allow connections to the Redis port (default 6379).
*   **Network Ports:** Ensure ports 7880 (Websocket) and potentially 7881 (WebRTC/TCP) and 50000-60000 (WebRTC/UDP) are open on the LiveKit server machine's firewall if needed for external access. The `--dev` flag simplifies this for local testing.

## 1. Install LiveKit CLI & Generate Keys

First, install the LiveKit command-line interface and generate the necessary API keys.

```bash
# Download and install the LiveKit CLI
curl -sSL https://get.livekit.io/cli | bash

# Generate API Key and Secret Key
# Make sure to save these securely! You will need them later.
livekit-server generate-keys


Take note of the generated API Key (e.g., APIxxxxxxxxxxxx) and API Secret (e.g., yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy). Store these securely.

2. Run the LiveKit Server

Start the main LiveKit server process. This command runs it in development mode (--dev), binds it to all network interfaces (--bind 0.0.0.0), and connects it to your Redis instance.

Replace <YOUR_API_KEY>, <YOUR_API_SECRET>, and <YOUR_REDIS_HOST:PORT> with your actual values.

livekit-server --keys "<YOUR_API_KEY>: <YOUR_API_SECRET>" \
               --dev \
               --redis-host=<YOUR_REDIS_HOST:PORT> \
               --bind 0.0.0.0
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

--keys: Provides the API key and secret for authentication.

--dev: Runs the server in development mode (simplifies TURN/networking, not recommended for production).

--redis-host: Specifies the address and port of your running Redis instance (e.g., 192.168.5.0:6379 or localhost:6379).

--bind 0.0.0.0: Makes the server listen on all available network interfaces.

Note: For long-running processes, consider running this using systemd, screen, or tmux.

3. Generate Participant Access Tokens

Participants need unique access tokens to join rooms. You typically generate these dynamically in your application's backend. Here's how to generate one manually using the CLI:

Replace <YOUR_API_KEY>, <YOUR_API_SECRET>, <ROOM_NAME>, and <PARTICIPANT_IDENTITY> with appropriate values.

lk token create \
  --api-key <YOUR_API_KEY> \
  --api-secret <YOUR_API_SECRET> \
  --join \
  --room <ROOM_NAME> \
  --identity <PARTICIPANT_IDENTITY> \
  --valid-for 24h
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

This token allows a user identified as <PARTICIPANT_IDENTITY> to join <ROOM_NAME>.

4. Frontend Integration

Use a LiveKit client SDK (JavaScript, React, iOS, Android, etc.) in your frontend application.

Provide the generated access token and the LiveKit server's WebSocket URL (e.g., ws://<YOUR_SERVER_IP>:7880) to the SDK to connect and join the video call.

5. Configure Egress Service

Egress is the LiveKit service responsible for recording or streaming rooms.

Create a Directory: Create a directory on the machine where you will run the Egress Docker container. This directory will hold the configuration and output recordings.

mkdir ~/livekit-egress
cd ~/livekit-egress
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Create config.yaml: Inside the ~/livekit-egress directory, create a file named config.yaml with the following content.

Replace <YOUR_API_KEY>, <YOUR_API_SECRET>, <YOUR_LIVEKIT_WS_URL>, and <YOUR_REDIS_HOST:PORT> with your actual values.

# ~/livekit-egress/config.yaml

log_level: debug  # Or info, warn, error
api_key: <YOUR_API_KEY>
api_secret: <YOUR_API_SECRET>

# Websocket URL of your LiveKit server
ws_url: <YOUR_LIVEKIT_WS_URL> # e.g., ws://localhost:7880 or ws://<your-server-ip>:7880

# Set to false if using self-signed certs or --dev mode on server
insecure: true

redis:
  # Address of your Redis instance (must be accessible from the Egress container)
  address: <YOUR_REDIS_HOST:PORT> # e.g., 192.168.5.0:6379 or host.docker.internal:6379 if Redis is on host
  # username: ""
  # password: ""
  # db: 0
  # use_tls: false
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Yaml
IGNORE_WHEN_COPYING_END
6. Run Egress Service (Docker)

Run the Egress service as a Docker container, mounting the configuration/output directory.

sudo docker run --rm \
  --cap-add SYS_ADMIN \
  -e EGRESS_CONFIG_FILE=/out/config.yaml \
  -v ~/livekit-egress:/out \
  livekit/egress
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

--rm: Removes the container when it stops.

--cap-add SYS_ADMIN: Needed for some Egress functionalities (like using a loopback device).

-e EGRESS_CONFIG_FILE=/out/config.yaml: Tells Egress where to find its config inside the container.

-v ~/livekit-egress:/out: Mounts the host directory ~/livekit-egress to /out inside the container. Config is read from here, and recordings are written here.

livekit/egress: The official Egress Docker image.

7. Trigger a Recording

To start recording a room, you need to send a request to the LiveKit server's API, which the Egress service listens for via Redis.

Add a Project (One-time setup per key pair): The lk CLI uses projects to group settings.

lk project add --api-key <YOUR_API_KEY> --api-secret <YOUR_API_SECRET> my_project_name --url http://localhost:7880
# Replace my_project_name with a name you like
# Replace http://localhost:7880 with your LiveKit server's HTTP URL (usually same host as ws://)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Create a Recording Request JSON: Create a file (e.g., composite.json) describing the recording parameters.

// composite.json
{
  "room_name": "critical_meet",
  "layout": "speaker-dark",
  "file_outputs": [
    {
      "filepath": "/out/recording-critical-meet.mp4"
    }
  ]
}
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Json
IGNORE_WHEN_COPYING_END

room_name: The name of the room to record.

layout: The visual layout (e.g., speaker-dark, grid-light).

filepath: The path inside the Egress container where the recording will be saved. Since /out is mapped to ~/livekit-egress on the host, this file will appear as ~/livekit-egress/recording-critical-meet.mp4.

Start the Egress Process: Use the lk CLI to send the start request. Make sure your project is selected or specify it.

# If you only have one project, this might work directly:
lk egress start --type room-composite composite.json

# Or specify the project:
# lk egress start --project my_project_name --type room-composite composite.json
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

The Egress service (running in Docker) will detect this request, join the specified room, and start recording to the specified file path.

8. Docker Permissions (Troubleshooting)

Sometimes, the Egress container might not have permission to write to the mounted host directory (~/livekit-egress). This happens if the user ID inside the container doesn't match the owner/permissions of the host directory.

Find the Egress Container ID:

sudo docker ps
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Look for the container using the livekit/egress image and note its CONTAINER ID.

Find the User ID inside the Container:

sudo docker exec -it <CONTAINER_ID> id
# Example Output: uid=1001(livekit) gid=0(root) groups=0(root),...
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Note the uid (e.g., 1001) and gid (e.g., 0).

Change Ownership of the Host Directory: On your host machine, change the owner of the output directory to match the user/group inside the container.

# Use the uid:gid found in the previous step
sudo chown <UID>:<GID> ~/livekit-egress
# Example: sudo chown 1001:0 ~/livekit-egress
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Now, the Egress container should be able to write the recording files into ~/livekit-egress.

You should now have a functional LiveKit server setup capable of hosting video calls and recording them using the Egress service. Remember to manage your API keys securely and adjust configuration for production environments (e.g., remove --dev, configure TURN properly, use systemd or similar for services).

IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
IGNORE_WHEN_COPYING_END
