# README

## Architecture

- **Frontend**: React.js
- **Backend**: Node.js API
- **Database**: PostgreSQL for data persistence

## Project Structure

```
├── backend/                 # Node.js API
│   ├── src/
│   │   ├── routes/         # API routes
│   │   ├── services/       # Business logic
│   │   └── server.js       # Express server
│   ├── Dockerfile
│   └── package.json
├── frontend/               # React application
│   ├── src/
│   │   ├── components/     # React components
│   │   ├── services/       # API services
│   │   └── tests/          # Component tests
│   ├── Dockerfile
│   └── package.json
```

## Getting Started

### Prerequisites

- Node.js 20
- Docker
- PostgreSQL (for local development)
- npm

### Local Development

#### Start PostgreSQL Container (Docker)

```
docker run -itd --rm --name devops-postgres \
  -e POSTGRES_DB=devops \
  -e POSTGRES_USER=devops \
  -e POSTGRES_PASSWORD=devops1234 \
  -p 5432:5432 postgres:15-alpine
```

##### PostgreSQL Details

```
POSTGRES_DB=devops
POSTGRES_USER=devops
POSTGRES_PASSWORD=devops1234
POSTGRES_HOSTNAME=localhost
```

> **⚠️ Command: To Remove PostgreSQL Container**: Delete it before docker compose up --build -d :
>
> ```bash
> docker stop devops-postgres
> ```

1. **Start Backend**:

```bash
cd backend
npm install
npm run migrate           # Run database migrations
npm run dev              # Start with auto-migration
```

2. **Start Frontend**:

```bash
cd frontend
npm install
npm start
```

Docker/Local Development

> **⚠️ Windows Users Only**: Run this command first to fix line endings:
>
> ```bash
> dos2unix backend/entrypoint.sh
> ```

> **⚠️ Remove PostgreSQL Container**: if already exists :
>
> ```bash
> docker stop devops-postgres
> ```

```bash
docker compose up 
```

## Environment Variables

### Backend

```
PORT=3001
NODE_ENV=development
DATABASE_URL=postgresql://devops_user:devops_password@localhost:5432/devops
```

### Frontend

```
REACT_APP_API_URL=http://localhost:3001/api
```
