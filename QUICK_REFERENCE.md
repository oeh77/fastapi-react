# FastAPI + React - Quick Feature Reference

A quick reference guide for developers using this cookiecutter template.

---

## 🚀 What's Included Out of the Box

### Backend (FastAPI)
✅ JWT Authentication  
✅ User CRUD Operations  
✅ PostgreSQL Database  
✅ SQLAlchemy ORM  
✅ Alembic Migrations  
✅ Celery Background Tasks  
✅ Redis Message Broker  
✅ Pytest Testing Framework  
✅ Auto-generated API Docs  
✅ Docker Compose Setup  

### Frontend (React + TypeScript)
✅ React 16+ with TypeScript  
✅ Material-UI Components  
✅ React Router v5  
✅ JWT Token Management  
✅ React-Admin Dashboard  
✅ Protected Routes  
✅ ESLint + Prettier  
✅ React Testing Library  

---

## 📋 Quick Commands

### Initial Setup
```bash
# Install cookiecutter
pip3 install cookiecutter

# Create new project
cookiecutter gh:Buuntu/fastapi-react

# Start project
cd your-project-name
chmod +x scripts/build.sh
./scripts/build.sh
```

### Development
```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend

# Rebuild after changes
docker-compose build backend
docker-compose build frontend
```

### Database
```bash
# Create migration
docker-compose exec backend alembic revision -m "description"

# Run migrations
docker-compose exec backend alembic upgrade head

# Rollback migration
docker-compose exec backend alembic downgrade -1
```

### Testing
```bash
# Run backend tests
docker-compose exec backend pytest

# Run specific test
docker-compose exec backend pytest app/tests/test_users.py

# Run with coverage
docker-compose exec backend pytest --cov=app

# Run frontend tests
docker-compose exec frontend npm test
```

---

## 🔐 Authentication Flow

### Login
```typescript
// Frontend
const response = await fetch('/api/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({ username: email, password })
});
const { access_token } = await response.json();
localStorage.setItem('token', access_token);
```

### Making Authenticated Requests
```typescript
// Frontend
const token = localStorage.getItem('token');
const response = await fetch('/api/v1/users/me', {
  headers: { 'Authorization': `Bearer ${token}` }
});
```

### Backend Protected Endpoint
```python
# Backend
from app.core.auth import get_current_active_user

@router.get("/protected")
async def protected_route(current_user=Depends(get_current_active_user)):
    return {"message": f"Hello {current_user.email}"}
```

---

## 🛣️ Available Routes

### Frontend Routes
| Route | Description | Auth Required |
|-------|-------------|---------------|
| `/` | Home page | No |
| `/login` | Login page | No |
| `/signup` | Signup page | No |
| `/logout` | Logout (clears token) | No |
| `/protected` | Example protected route | Yes |
| `/admin` | Admin dashboard | Yes (Superuser) |

### Backend API Endpoints
| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| POST | `/api/token` | Login | No |
| POST | `/api/signup` | Signup | No |
| GET | `/api/v1` | Hello World | No |
| GET | `/api/v1/users` | List all users | Superuser |
| GET | `/api/v1/users/me` | Get current user | User |
| GET | `/api/v1/users/{id}` | Get user by ID | Superuser |
| POST | `/api/v1/users` | Create user | Superuser |
| PUT | `/api/v1/users/{id}` | Update user | Superuser |
| DELETE | `/api/v1/users/{id}` | Delete user | Superuser |
| GET | `/api/docs` | API Documentation | No |
| GET | `/api/v1/task` | Example Celery task | No |

---

## 🧪 Testing Utilities

### Backend Fixtures (Pytest)
```python
# Auto-imported from conftest.py

def test_example(test_db, client, test_user, user_token_headers):
    # test_db: Database session
    # client: TestClient for API requests
    # test_user: Regular user instance
    # user_token_headers: Auth headers for regular user
    
    response = client.get("/api/v1/users/me", headers=user_token_headers)
    assert response.status_code == 200
```

### Available Fixtures
- `test_db`: Empty test database session
- `test_user`: Regular user (fake@email.com)
- `test_superuser`: Admin user (fakeadmin@email.com)
- `client`: TestClient instance
- `user_token_headers`: Auth headers for regular user
- `superuser_token_headers`: Auth headers for superuser
- `test_password`: Default password ("securepassword")

---

## 📦 Adding New Features

### Add New API Endpoint

1. **Create Router** (if needed)
```python
# backend/app/api/api_v1/routers/items.py
from fastapi import APIRouter
items_router = r = APIRouter()

@r.get("/items")
async def list_items():
    return {"items": []}
```

2. **Register Router**
```python
# backend/app/main.py
from app.api.api_v1.routers.items import items_router

app.include_router(items_router, prefix="/api/v1", tags=["items"])
```

3. **Add Tests**
```python
# backend/app/api/api_v1/routers/tests/test_items.py
def test_list_items(client):
    response = client.get("/api/v1/items")
    assert response.status_code == 200
```

### Add New Database Model

1. **Define Model**
```python
# backend/app/db/models.py
class Item(Base):
    __tablename__ = "item"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    user_id = Column(Integer, ForeignKey("user.id"))
```

2. **Create Schema**
```python
# backend/app/db/schemas.py
class ItemBase(BaseModel):
    name: str

class ItemCreate(ItemBase):
    pass

class Item(ItemBase):
    id: int
    user_id: int
    
    class Config:
        orm_mode = True
```

3. **Create Migration**
```bash
docker-compose exec backend alembic revision -m "add_item_table"
# Edit migration file
docker-compose exec backend alembic upgrade head
```

### Add New React Page

1. **Create Component**
```typescript
// frontend/src/views/NewPage.tsx
import React from 'react';

export const NewPage: React.FC = () => {
  return <div>New Page</div>;
};
```

2. **Export from views**
```typescript
// frontend/src/views/index.ts
export { NewPage } from './NewPage';
```

3. **Add Route**
```typescript
// frontend/src/Routes.tsx
import { NewPage } from './views';

// In Routes component:
<Route path="/new-page" component={NewPage} />
```

---

## 🎨 Admin Dashboard Customization

### Add Resource to Admin
```typescript
// frontend/src/admin/Admin.tsx
import { ItemList, ItemEdit, ItemCreate } from './Items';

<ReactAdmin dataProvider={dataProvider} authProvider={authProvider}>
  {(permissions: 'admin' | 'user') => [
    permissions === 'admin' ? (
      <>
        <Resource name="users" list={UserList} edit={UserEdit} create={UserCreate} />
        <Resource name="items" list={ItemList} edit={ItemEdit} create={ItemCreate} />
      </>
    ) : null,
  ]}
</ReactAdmin>
```

### Create List Component
```typescript
// frontend/src/admin/Items/ItemList.tsx
import React from 'react';
import { List, Datagrid, TextField, EditButton } from 'react-admin';

export const ItemList = (props: any) => (
  <List {...props}>
    <Datagrid>
      <TextField source="id" />
      <TextField source="name" />
      <EditButton />
    </Datagrid>
  </List>
);
```

---

## 🔧 Common Configuration

### Environment Variables

**Backend** (docker-compose.yml):
```yaml
environment:
  DATABASE_URL: postgresql://user:pass@postgres:5432/db
  SECRET_KEY: your_secret_key
  PYTHONPATH: .
```

**Frontend** (docker-compose.yml):
```yaml
environment:
  NODE_ENV: development
  REACT_APP_API_URL: http://localhost:8000
```

### Database Connection
```python
# backend/app/core/config.py
SQLALCHEMY_DATABASE_URI = os.getenv(
    "DATABASE_URL",
    "postgresql://postgres:password@localhost/app"
)
```

### JWT Configuration
```python
# backend/app/core/security.py
SECRET_KEY = "your_secret_key"  # Change in production!
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
```

---

## 🐛 Debugging

### Backend Debugging
```python
# Add to any endpoint for debugging
import ipdb; ipdb.set_trace()

# Or use print debugging
from app.core.logging_config import logger
logger.debug(f"Debug info: {variable}")
```

### Frontend Debugging
```typescript
// Use React DevTools browser extension
console.log('Debug:', data);

// Add breakpoint in browser DevTools
debugger;
```

### Database Debugging
```bash
# Access PostgreSQL
docker-compose exec postgres psql -U postgres -d app

# List tables
\dt

# Query users
SELECT * FROM "user";
```

---

## 📊 Monitoring

### Service URLs (Default)
- **Frontend**: http://localhost:8000
- **Backend API**: http://localhost:8000/api
- **API Docs**: http://localhost:8000/api/docs
- **Admin Dashboard**: http://localhost:8000/admin
- **Flower (Celery)**: http://localhost:5555
- **PostgreSQL**: localhost:5432
- **Redis**: localhost:6379

### Check Service Health
```bash
# Check running containers
docker-compose ps

# Check logs
docker-compose logs -f [service_name]

# Check backend health
curl http://localhost:8000/api/v1

# Check database connection
docker-compose exec postgres pg_isready
```

---

## 🚨 Common Issues

### Issue: Nginx 502 Bad Gateway
**Cause**: Frontend webpack server not ready yet  
**Solution**: Wait a minute for webpack to compile, then refresh

### Issue: Database connection error
**Cause**: PostgreSQL container not ready  
**Solution**: Wait for postgres to fully start: `docker-compose logs postgres`

### Issue: Permission denied on scripts
**Cause**: Build script not executable  
**Solution**: `chmod +x scripts/build.sh`

### Issue: Port already in use
**Cause**: Another service using port 8000  
**Solution**: Change port in cookiecutter.json or stop conflicting service

### Issue: Frontend hot reload not working
**Cause**: Docker volume issues  
**Solution**: Restart frontend container: `docker-compose restart frontend`

---

## 📚 Key Files

### Backend
| File | Purpose |
|------|---------|
| `backend/app/main.py` | FastAPI app entry point |
| `backend/app/core/config.py` | Configuration settings |
| `backend/app/core/security.py` | JWT and password hashing |
| `backend/app/core/auth.py` | Authentication logic |
| `backend/app/db/models.py` | SQLAlchemy models |
| `backend/app/db/schemas.py` | Pydantic schemas |
| `backend/app/db/crud.py` | Database operations |
| `backend/conftest.py` | Pytest fixtures |

### Frontend
| File | Purpose |
|------|---------|
| `frontend/src/index.tsx` | React entry point |
| `frontend/src/App.tsx` | Main App component |
| `frontend/src/Routes.tsx` | Route definitions |
| `frontend/src/utils/auth.ts` | Auth utilities |
| `frontend/src/admin/Admin.tsx` | Admin dashboard |
| `frontend/package.json` | Dependencies |

### DevOps
| File | Purpose |
|------|---------|
| `docker-compose.yml` | Container orchestration |
| `nginx/nginx.conf` | Reverse proxy config |
| `scripts/build.sh` | Initial build script |
| `.gitignore` | Git ignore rules |

---

## 🎓 Learning Resources

### FastAPI
- Official Docs: https://fastapi.tiangolo.com/
- Tutorial: https://fastapi.tiangolo.com/tutorial/

### React
- Official Docs: https://reactjs.org/
- TypeScript Handbook: https://www.typescriptlang.org/docs/

### React-Admin
- Official Docs: https://marmelab.com/react-admin/
- Tutorial: https://marmelab.com/react-admin/Tutorial.html

### SQLAlchemy
- Official Docs: https://docs.sqlalchemy.org/
- ORM Tutorial: https://docs.sqlalchemy.org/en/14/orm/tutorial.html

### Docker
- Docker Docs: https://docs.docker.com/
- Docker Compose: https://docs.docker.com/compose/

---

## 💡 Tips & Best Practices

### Security
✅ Always use HTTPS in production  
✅ Change default SECRET_KEY  
✅ Use strong passwords  
✅ Keep dependencies updated (Dependabot active)  
✅ Validate all inputs  
✅ Use environment variables for secrets  

### Development
✅ Write tests for new features  
✅ Use type hints in Python  
✅ Use TypeScript types in React  
✅ Follow ESLint/Prettier rules  
✅ Keep components small and focused  
✅ Use database migrations for schema changes  

### Performance
✅ Use database indexes appropriately  
✅ Implement pagination for large datasets  
✅ Use Celery for long-running tasks  
✅ Cache frequently accessed data  
✅ Optimize database queries  
✅ Use React.memo for expensive components  

---

## 🤝 Getting Help

1. Check the main README.md
2. Review API docs at `/api/docs`
3. Check GitHub issues: https://github.com/Buuntu/fastapi-react/issues
4. Read CONTRIBUTING.md for contribution guidelines

---

*Quick Reference v1.0 - For FastAPI + React Cookiecutter Template*
