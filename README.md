# 📄 Documentation – Web Server Configuration

## 1. Keyboard Configuration

### 📌 Objective:
Configure the keyboard to match the layout used (e.g., AZERTY for France).

### 🧰 Command:

```bash
sudo dpkg-reconfigure keyboard-configuration
```

### 📝 Explanation:

This command launches an interactive tool to select the keyboard type and desired layout. Follow the on-screen steps to choose:
The keyboard model (e.g., "Generic 105-key (Intl) PC")
The country and layout (e.g., "French")
Special keys (compose, Alt key, etc.)


## 2. System Update and Upgrade

### 🧰 Command:

```bash
sudo apt update && sudo apt upgrade -y
```

### 📝 Explanation:

apt update: Updates the list of available packages.
apt upgrade -y: Installs the latest available versions of the packages already installed, without manual confirmation due to -y.

## 3. NGINX Installation

### 🧰 Commands:

```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

### 📝 Explanation:
apt install nginx -y: Installs the NGINX web server.
systemctl start nginx: Starts the NGINX service immediately.
systemctl enable nginx: Enables NGINX to start automatically at each VM boot.

## 4. VM Network Configuration (VirtualBox)

### 🎯 Objective:

Access your web server from your host machine.
🔧 NAT (default)
The VM is isolated, but you can access it via port forwarding.
🔄 Steps:
1. Shut down the VM if it is running.
2. Go to VirtualBox > VM Settings > Network > Advanced > Port Forwarding.
3. Add a rule as follows:
   
| Field       | Value     |
|-------------|-----------|
| Protocol    | TCP       |
| Host IP     | 127.0.0.1 |
| Host Port   | 8080      |
| Guest IP    | (leave empty) |
| Guest Port  | 80        |

💡 Example:
Access the NGINX server via http://127.0.0.1:8080 on your host machine.
