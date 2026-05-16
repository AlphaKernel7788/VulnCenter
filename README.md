# n8n Vulnerability Management System

This project is a vulnerability management system built with n8n and PostgreSQL. It includes a dashboard for visualizing vulnerability data and automated workflows for processing security information.

## Project Structure

```
├── backup/
│   └── docker-compose.yml     # Docker Compose configuration
├── postgres-init/
│   └── init.sql               # PostgreSQL initialization script
├── n8n-data/                  # n8n persistent data
│   └── nodes/                 # Custom n8n nodes (currently empty)
└── index.html                 # Vulnerability dashboard frontend
```

## Services

### PostgreSQL Database
- **Image**: postgres:15-alpine
- **Database Name**: vulnmgmt
- **Username**: n8n
- **Password**: test123
- **Port**: 5432 (default)

### n8n Workflow Engine
- **Image**: n8nio/n8n:latest
- **Port**: 5678
- **Database Connection**: PostgreSQL backend

## Setup Instructions

1. Clone this repository
2. Run `docker-compose up -d` from the backup directory
3. Access n8n at http://localhost:5678
4. Access the vulnerability dashboard by opening `index.html` in a browser

## Features

- Automated vulnerability data processing workflows
- Dashboard visualization with Chart.js
- PostgreSQL backend for data persistence
- Team-based vulnerability ownership tracking

## Configuration

The system is configured to:
- Use PostgreSQL as the database backend
- Create a dedicated `vulnmgmt` database
- Provide a web-based dashboard for vulnerability visualization
- Support team-based filtering (Linux Team, Windows Team)

## API Endpoints

The dashboard expects vulnerability data from:
- `http://localhost:8080/api/vulns` - Vulnerability API endpoint

## Security Notes

⚠️ **Warning**: The default credentials are used in this setup:
- PostgreSQL: username `n8n`, password `test123`
- These should be changed in production environments

## Customization

To customize this setup:
1. Modify the `docker-compose.yml` file to change service configurations
2. Update `init.sql` to create additional databases or tables
3. Modify `index.html` to customize the dashboard
4. Add custom n8n workflows to process vulnerability data

## Data Persistence

All data is persisted in:
- `./postgres-data` - PostgreSQL data
- `./n8n-data` - n8n configuration and workflow data
