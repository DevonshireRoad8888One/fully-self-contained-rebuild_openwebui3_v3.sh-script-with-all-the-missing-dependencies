Just paste the code block into the Ubuntu terminal and Open Web.UI A.I. Webserver will be installed.
At this point go to http://127.0.0.1:8001/
and you will create your Open Web.UI account, and thats it...   :)

The command to start the server from the terminal after the install is done, 

cd ~/openwebui3 && ./start.sh





# fully-self-contained-rebuild_openwebui3_v3.sh-script-with-all-the-missing-dependencies
fully self-contained rebuild_openwebui3_v3.sh script with all the missing dependencies added for a brand new Ubuntu installation.
Here is your improved, fully self-contained rebuild_openwebui3_v3.sh script with all the missing dependencies added for a brand new Ubuntu installation.Key improvements:Full Python 3.12 setup via deadsnakes PPA (required on fresh Ubuntu)
Installs zstd, ffmpeg, and all other common missing packages
Automatically installs Ollama if not present
Better ordering and idempotency
More robust error handling

New Script (Copy & Run)bash

cat > "$HOME/rebuild_openwebui3_v3.sh" <<'REBUILD3'
#!/usr/bin/env bash
set -euo pipefail

cd "$HOME"

BASE="$HOME/openwebui3"
WEBUI_PORT="8001"
OLLAMA_PORT="11436"
OPENWEBUI_VERSION="0.9.6"

echo "=================================================="
echo " Open WebUI Test Bench #3 Rebuild Script v3"
echo " Folder:       $BASE"
echo " WebUI Port:   $WEBUI_PORT"
echo " Ollama Port:  $OLLAMA_PORT"
echo "=================================================="

echo "🔧 Updating system and installing all required dependencies..."

sudo apt update
sudo apt upgrade -y

# Essential tools for adding PPAs
sudo apt install -y software-properties-common curl ca-certificates gnupg lsb-release

# Add deadsnakes PPA for Python 3.12
echo "🐍 Adding deadsnakes PPA for Python 3.12..."
sudo add-apt-repository ppa:deadsnakes/ppa -y

sudo apt update

# Install Python 3.12 and related packages
sudo apt install -y python3.12 python3.12-dev python3.12-venv python3-pip

# Additional system dependencies
sudo apt install -y \
    python3 python3-pip python3-venv \
    curl ca-certificates psmisc iproute2 \
    zstd ffmpeg \
    build-essential pkg-config

echo "✅ System dependencies installed."

# Install Ollama if not present
if ! command -v ollama >/dev/null 2>&1; then
    echo "🧠 Installing Ollama..."
    curl -fsSL https://ollama.com/install.sh | sh
    # Give it a moment to finish
    sleep 3
else
    echo "🧠 Ollama is already installed."
fi

echo "📁 Creating fresh isolated folder structure..."
rm -rf "$BASE"
mkdir -p "$BASE"/{data,models,logs,run}

echo "🐍 Creating Python virtual environment with Python 3.12..."
python3.12 -m venv "$BASE/env1"

echo "⬆️ Upgrading pip tools..."
cd "$BASE"
source "$BASE/env1/bin/activate"
python -m pip install --upgrade pip setuptools wheel

echo "🌐 Installing Open WebUI..."
if ! pip install "open-webui==$OPENWEBUI_VERSION"; then
  echo "⚠️ Specific version failed. Installing latest open-webui..."
  pip install open-webui
fi

deactivate

echo "🔐 Creating isolated .env file..."
SECRET="$(python3 - <<'PY'
import secrets
print(secrets.token_urlsafe(48))
PY
)"

cat > "$BASE/.env" <<ENV
WEBUI_PORT=$WEBUI_PORT
OLLAMA_PORT=$OLLAMA_PORT
WEBUI_SECRET_KEY=$SECRET
ENV

chmod 600 "$BASE/.env"

echo "📝 Creating fixed start.sh..."
cat > "$BASE/start.sh" <<'START3'
#!/usr/bin/env bash
set -e

BASE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

if [ -f "$BASE/.env" ]; then
  set -a
  . "$BASE/.env"
  set +a
fi

WEBUI_PORT="${WEBUI_PORT:-8001}"
OLLAMA_PORT="${OLLAMA_PORT:-11436}"

WEBUI_URL="http://127.0.0.1:$WEBUI_PORT"
OLLAMA_URL="http://127.0.0.1:$OLLAMA_PORT"

pids_on_port() {
  ss -ltnp "sport = :$1" 2>/dev/null | grep -o 'pid=[0-9]\+' | cut -d= -f2 | sort -u || true
}

echo "🚀 Starting ISOLATED Open WebUI Test Bench #3"
echo "🌐 Open WebUI: $WEBUI_URL"
echo "🧠 Ollama:     $OLLAMA_URL"
echo "📁 Folder:     $BASE"

if [ ! -d "$BASE/env1" ] || [ ! -x "$BASE/env1/bin/open-webui" ]; then
  echo "❌ Missing Python virtual environment or open-webui executable."
  echo "Run the installer again."
  exit 1
fi

if ! command -v ollama >/dev/null 2>&1; then
  echo "❌ Ollama command not found."
  exit 1
fi

mkdir -p "$BASE/data" "$BASE/models" "$BASE/logs" "$BASE/run"

# Start isolated Ollama if needed
if curl -fsS "$OLLAMA_URL/api/version" >/dev/null 2>&1; then
  echo "✅ Isolated Ollama #3 already running"
else
  if [ -n "$(pids_on_port "$OLLAMA_PORT")" ]; then
    echo "❌ Port $OLLAMA_PORT is in use by another process."
    exit 1
  fi

  echo "🔄 Starting isolated Ollama #3..."
  (
    export OLLAMA_HOST="127.0.0.1:$OLLAMA_PORT"
    export OLLAMA_MODELS="$BASE/models"
    nohup ollama serve > "$BASE/ollama.log" 2>&1 &
    echo $! > "$BASE/ollama.pid"
  )

  for i in $(seq 1 30); do
    if curl -fsS "$OLLAMA_URL/api/version" >/dev/null 2>&1; then
      echo "✅ Ollama #3 ready"
      break
    fi
    sleep 1
  done
fi

# Start Open WebUI
if curl -fsS "$WEBUI_URL/api/version" >/dev/null 2>&1; then
  echo "✅ Open WebUI #3 already running"
  exit 0
fi

if [ -n "$(pids_on_port "$WEBUI_PORT")" ]; then
  echo "❌ Port $WEBUI_PORT is already in use."
  exit 1
fi

echo "🚀 Starting Open WebUI #3..."
source "$BASE/env1/bin/activate"

export DATA_DIR="$BASE/data"
export WEBUI_SECRET_KEY="${WEBUI_SECRET_KEY:-openwebui3-test-secret}"
export OLLAMA_BASE_URL="$OLLAMA_URL"
export OLLAMA_API_BASE_URL="$OLLAMA_URL"
export OLLAMA_BASE_URLS="$OLLAMA_URL"
export HOST="127.0.0.1"
export PORT="$WEBUI_PORT"

cd "$BASE"
exec "$BASE/env1/bin/open-webui" serve --host 127.0.0.1 --port "$WEBUI_PORT"
START3

echo "📝 Creating stop.sh..."
cat > "$BASE/stop.sh" <<'STOP3'
#!/usr/bin/env bash
set -e
BASE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

if [ -f "$BASE/.env" ]; then
  set -a; . "$BASE/.env"; set +a
fi

WEBUI_PORT="${WEBUI_PORT:-8001}"
OLLAMA_PORT="${OLLAMA_PORT:-11436}"

pids_on_port() {
  ss -ltnp "sport = :$1" 2>/dev/null | grep -o 'pid=[0-9]\+' | cut -d= -f2 | sort -u || true
}

kill_pids() {
  NAME="$1"; PIDS="$2"
  [ -z "$PIDS" ] && return
  echo "🛑 Stopping $NAME: $PIDS"
  for PID in $PIDS; do kill "$PID" 2>/dev/null || true; done
  sleep 2
  for PID in $PIDS; do
    if kill -0 "$PID" 2>/dev/null; then kill -9 "$PID" 2>/dev/null || true; fi
  done
}

echo "🛑 Stopping Open WebUI Test Bench #3..."
kill_pids "Open WebUI #3" "$(pids_on_port "$WEBUI_PORT")"

OLLAMA_PIDS="$(pids_on_port "$OLLAMA_PORT")"
if [ -z "$OLLAMA_PIDS" ] && [ -f "$BASE/ollama.pid" ]; then
  PID_FROM_FILE="$(cat "$BASE/ollama.pid" 2>/dev/null || true)"
  [ -n "$PID_FROM_FILE" ] && kill -0 "$PID_FROM_FILE" 2>/dev/null && OLLAMA_PIDS="$PID_FROM_FILE"
fi
kill_pids "Ollama #3" "$OLLAMA_PIDS"

rm -f "$BASE/ollama.pid"
echo "✅ Stopped."
STOP3

chmod +x "$BASE/start.sh" "$BASE/stop.sh"

echo "=================================================="
echo "✅ Open WebUI Test Bench #3 rebuilt successfully!"
echo "🌐 WebUI:   http://127.0.0.1:$WEBUI_PORT"
echo "🧠 Ollama:  http://127.0.0.1:$OLLAMA_PORT"
echo "=================================================="

echo "🚀 Starting OpenWebUI3 now..."
cd "$BASE"
./start.sh
REBUILD3

chmod +x "$HOME/rebuild_openwebui3_v3.sh"
"$HOME/rebuild_openwebui3_v3.sh"

How to use:Just run this single command on the new Ubuntu machine:bash

bash -c "$(curl -fsSL https://your-script-url-if-you-host-it OR just paste the whole thing)"

Or copy the entire block above into a terminal.

