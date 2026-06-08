# Setup and Installation Guide

## Prerequisites

- Docker & Docker Compose
- Git
- Node.js 18+ (for local development)
- Python 3.11+ (for local development)
- PostgreSQL client tools (optional)

## Quick Start with Docker

### 1. Clone the Repository

```bash
git clone https://github.com/visiotechnology26-hash/School_admission_system.git
cd School_admission_system
```

### 2. Setup Environment Variables

```bash
# Copy environment examples
cp .env.example .env
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Edit the `.env` files and update the values as needed.

### 3. Start Services

```bash
# Build and start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

### 4. Access the Application

- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:5000/api
- **Database**: localhost:5432
- **Redis**: localhost:6379

## Local Development Setup

### Backend Setup

```bash
cd backend

# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Windows:
venv\Scripts\activate
# On macOS/Linux:
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Create .env file
cp .env.example .env

# Initialize database
flask db init
flask db migrate
flask db upgrade

# Run development server
python -m flask run
```

### Frontend Setup

```bash
cd frontend

# Install dependencies
npm install

# Create .env file
cp .env.example .env

# Start development server
npm start
```

## Database Initialization

The database is automatically initialized when using Docker Compose. For manual initialization:

```bash
# Connect to PostgreSQL
psql -h localhost -U admin -d admissions_system

# Run init.sql
\i backend/database/init.sql
```

## API Documentation

### Authentication Endpoints

#### Register User
```
POST /api/auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123",
  "first_name": "John",
  "last_name": "Doe"
}
```

#### Login
```
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}

Response:
{
  "access_token": "jwt_token_here",
  "user": {
    "id": 1,
    "email": "user@example.com",
    "role": "applicant"
  }
}
```

#### Get Current User
```
GET /api/auth/me
Authorization: Bearer {access_token}
```

### Schools Endpoints

#### Get All Schools
```
GET /api/schools
```

#### Get School Details
```
GET /api/schools/{school_id}
```

#### Create School (Admin Only)
```
POST /api/schools
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "name": "Premier SHS",
  "code": "PSHS001",
  "type": "SHS",
  "email": "info@premier.edu.gh",
  "phone": "+233123456789",
  "address": "123 Main St",
  "city": "Accra",
  "region": "Greater Accra"
}
```

### Applications Endpoints

#### Get Applications
```
GET /api/applications
Authorization: Bearer {access_token}
```

#### Create Application
```
POST /api/applications
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "bece_index": "1234567890",
  "school_id": 1,
  "form_data": {
    "field1": "value1"
  }
}
```

#### Submit Application
```
POST /api/applications/{app_id}/submit
Authorization: Bearer {access_token}
```

### Documents Endpoints

#### Upload Document
```
POST /api/documents/upload
Authorization: Bearer {access_token}
Content-Type: multipart/form-data

- file: <document_file>
- application_id: 1
- document_type: "placement_letter"
```

#### Verify Document (Reviewer Only)
```
POST /api/documents/{doc_id}/verify
Authorization: Bearer {access_token}
```

### Payments Endpoints

#### Initiate Payment
```
POST /api/payments
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "application_id": 1,
  "amount": 100.00,
  "payment_method": "mobile_money"
}
```

#### Confirm Payment
```
POST /api/payments/{payment_id}/confirm
Authorization: Bearer {access_token}
```

## Admin Functions

### Access Admin Panel
Navigate to `/admin` (Admin role required)

### Create Admin User

```bash
# Use Flask shell
flask shell

# Create admin user
from models import User, UserRole
from app import db

admin = User(
    email="admin@example.com",
    first_name="Admin",
    last_name="User",
    role=UserRole.ADMIN
)
admin.set_password("admin_password")
db.session.add(admin)
db.session.commit()
```

### View Audit Logs
```
GET /api/admin/audit-logs
Authorization: Bearer {access_token}
```

### Get Dashboard Stats
```
GET /api/admin/dashboard/stats
Authorization: Bearer {access_token}
```

## Troubleshooting

### Port Already in Use

```bash
# Find process using port
lsof -i :5000  # Backend
lsof -i :3000  # Frontend

# Kill process
kill -9 <PID>
```

### Database Connection Error

```bash
# Check PostgreSQL is running
docker-compose logs postgres

# Verify connection string in .env
# Format: postgresql://user:password@host:port/database
```

### Docker Build Issues

```bash
# Clear Docker cache
docker-compose down -v
docker system prune -a

# Rebuild
docker-compose up -d --build
```

## Deployment

### Production Checklist

- [ ] Update `SECRET_KEY` in `.env`
- [ ] Update `JWT_SECRET_KEY`
- [ ] Set `FLASK_ENV=production`
- [ ] Set `DEBUG=False`
- [ ] Configure email settings
- [ ] Setup payment gateway API keys
- [ ] Configure AWS S3 for file storage (optional)
- [ ] Setup SSL/TLS certificates
- [ ] Configure domain name
- [ ] Setup backup strategy for database

### Deploy to Cloud

#### Heroku
```bash
heroku login
heroku create your-app-name
heroku config:set FLASK_ENV=production
git push heroku main
```

#### AWS/Azure
```bash
# Build Docker images
docker build -t admissions-backend:latest ./backend
docker build -t admissions-frontend:latest ./frontend

# Push to container registry
docker tag admissions-backend:latest <registry>/admissions-backend:latest
docker push <registry>/admissions-backend:latest
```

## Development Guidelines

### Code Style

- Python: PEP 8 (use `black` for formatting)
- JavaScript/TypeScript: ESLint + Prettier
- React: Functional components with hooks

### Running Tests

```bash
# Backend tests
cd backend
pytest

# Frontend tests
cd frontend
npm test
```

### Git Workflow

```bash
# Create feature branch
git checkout -b feature/feature-name

# Make changes and commit
git add .
git commit -m "Add feature description"

# Push and create pull request
git push origin feature/feature-name
```

## Support

For issues or questions:
- Create an issue on GitHub
- Contact: support@admissionssystem.com

## License

This project is licensed under MIT License - see LICENSE file for details.
