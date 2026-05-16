# Vulnerability Management System - Setup Guide

This guide provides step-by-step instructions to replicate the n8n-based vulnerability management system on a Debian-based OS (Debian, Ubuntu, etc.).

## Prerequisites

Before starting, ensure your system is up-to-date:

```bash
sudo apt update && sudo apt upgrade -y
```

## Required Software Installation

### 1. Install Docker and Docker Compose

```bash
# Install Docker
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release -y
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add your user to the docker group
sudo usermod -aG docker $USER
```

**Note**: Log out and log back in for the group changes to take effect.

### 2. Install Additional Tools

```bash
# Install basic tools
sudo apt install git curl wget -y
```

## Project Setup

### 1. Create Project Directory Structure

```bash
# Create the main project directory
mkdir -p ~/vulnmgmt/{backup,postgres-init,n8n-data/nodes}

# Navigate to the project directory
cd ~/vulnmgmt
```

### 2. Create Docker Compose File

Create `backup/docker-compose.yml`:

```bash
mkdir -p backup
cat > backup/docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=test123
      - POSTGRES_DB=n8n
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n"]
      interval: 5s
      timeout: 5s
      retries: 5

  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=test123
      - DB_POSTGRESDB_DATABASE=n8n
      - GENERIC_TIMEZONE=UTC
    volumes:
      - ./n8n-data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
EOF
```

### 3. Create PostgreSQL Initialization Script

```bash
mkdir -p postgres-init
cat > postgres-init/init.sql << 'EOF'
CREATE DATABASE vulnmgmt;
EOF
```

### 4. Create n8n Nodes Directory Structure

```bash
mkdir -p n8n-data/nodes
cat > n8n-data/nodes/package.json << 'EOF'
{
  "name": "installed-nodes",
  "private": true,
  "dependencies": {}
}
EOF
```

### 5. Create Vulnerability Dashboard

Create `index.html`:

```bash
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>

<h2>Vulnerability Dashboard</h2>

<select id="ownerFilter">
  <option value="">All</option>
  <option>Linux Team</option>
  <option>Windows Team</option>
</select>

<canvas id="chart"></canvas>

<script>
async function loadData() {
  const res = await fetch("http://localhost:8080/api/vulns");
  const data = await res.json();

  const counts = {};

  data.forEach(v => {
    counts[v.owner] = (counts[v.owner] || 0) + 1;
  });

  new Chart(document.getElementById("chart"), {
    type: "pie",
    data: {
      labels: Object.keys(counts),
      datasets: [{ data: Object.values(counts) }]
    }
  });
}

loadData();
</script>

</body>
</html>
EOF
```

## Running the System

### 1. Start the Services

```bash
# Navigate to the backup directory where docker-compose.yml is located
cd ~/vulnmgmt/backup

# Start the services
docker-compose up -d
```

### 2. Verify Services are Running

```bash
# Check if services are running
docker-compose ps

# Check logs if needed
docker-compose logs -f
```

### 3. Access the Applications

- **n8n**: http://localhost:5678
- **Vulnerability Dashboard**: Open `~/vulnmgmt/index.html` in a web browser

## Useful Commands

### Docker Management

```bash
# Stop services
docker-compose down

# Restart services
docker-compose restart

# View logs
docker-compose logs

# View logs for a specific service
docker-compose logs n8n
docker-compose logs postgres

# Execute commands in containers
docker-compose exec postgres bash
docker-compose exec n8n bash
```

### Database Access

```bash
# Access PostgreSQL database
docker-compose exec postgres psql -U n8n -d n8n

# Access vulnmgmt database
docker-compose exec postgres psql -U n8n -d vulnmgmt
```

### Data Backup and Restore

```bash
# Backup PostgreSQL data
docker-compose exec postgres pg_dump -U n8n n8n > backup_n8n_$(date +%Y%m%d).sql
docker-compose exec postgres pg_dump -U n8n vulnmgmt > backup_vulnmgmt_$(date +%Y%m%d).sql

# Restore PostgreSQL data (example)
# docker-compose exec -T postgres psql -U n8n n8n < backup_n8n_20230101.sql
```

## Troubleshooting

### Common Issues

1. **Permission denied errors with Docker**:
   - Ensure your user is in the docker group
   - Log out and log back in after adding user to docker group

2. **Port conflicts**:
   - Change the ports in docker-compose.yml if 5678 is already in use

3. **Database connection issues**:
   - Check that the postgres service is healthy:
     ```bash
     docker-compose ps
     ```

4. **Dashboard not loading data**:
   - Ensure the API endpoint (http://localhost:8080/api/vulns) is accessible
   - Check browser console for CORS or network errors

### Checking Service Health

```bash
# Check if PostgreSQL is ready
docker-compose exec postgres pg_isready -U n8n

# Check container logs
docker-compose logs postgres
docker-compose logs n8n
```

## Customization

### Adding Custom n8n Nodes

1. Navigate to the nodes directory:
   ```bash
   cd ~/vulnmgmt/n8n-data/nodes
   ```

2. Install custom nodes:
   ```bash
   # Example: Install a custom node package
   npm install <custom-node-package>
   ```

3. Restart n8n:
   ```bash
   cd ~/vulnmgmt/backup
   docker-compose restart n8n
   ```

### Modifying the Dashboard

Edit `~/vulnmgmt/index.html` to customize the dashboard according to your needs.

### Changing Database Credentials

1. Update `backup/docker-compose.yml` with new credentials
2. Update `postgres-init/init.sql` if needed
3. Restart the services:
   ```bash
   docker-compose down
   docker-compose up -d
   ```

## Data Persistence

All data is persisted in:
- `~/vulnmgmt/backup/postgres-data` - PostgreSQL data
- `~/vulnmgmt/n8n-data` - n8n configuration and workflow data

To backup the entire system:
```bash
# Create a backup of the entire project
tar -czf vulnmgmt-backup-$(date +%Y%m%d).tar.gz ~/vulnmgmt
```

## Security Considerations

⚠️ **Warning**: This setup uses default credentials:
- PostgreSQL: username `n8n`, password `test123`

For production use, you should:
1. Change the default credentials in `docker-compose.yml`
2. Use environment variables or Docker secrets for sensitive information
3. Implement proper firewall rules
4. Use HTTPS for web interfaces
5. Regularly update Docker images

To change credentials:
```bash
# Edit the docker-compose.yml file
nano backup/docker-compose.yml

# Change these environment variables:
# POSTGRES_PASSWORD=your_secure_password
# DB_POSTGRESDB_PASSWORD=your_secure_password

# Then restart the services
docker-compose down
docker-compose up -d
```