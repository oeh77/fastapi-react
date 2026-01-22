# Implementation Guide - Top Priority Features

This guide provides actionable steps to implement the most important missing features identified in the feature review.

---

## 1. Email Functionality

### Backend Implementation

#### Step 1: Add Dependencies
```bash
# Add to backend/requirements.txt
python-multipart==0.0.5  # Already included
emails==0.6
jinja2==2.11.3  # Already included
aiosmtplib==1.1.6  # For async email sending
```

#### Step 2: Create Email Configuration
```python
# backend/app/core/config.py
# Add these settings
SMTP_HOST: str = os.getenv("SMTP_HOST", "smtp.gmail.com")
SMTP_PORT: int = int(os.getenv("SMTP_PORT", "587"))
SMTP_USER: str = os.getenv("SMTP_USER", "")
SMTP_PASSWORD: str = os.getenv("SMTP_PASSWORD", "")
EMAILS_FROM_EMAIL: str = os.getenv("EMAILS_FROM_EMAIL", "noreply@example.com")
EMAILS_FROM_NAME: str = os.getenv("EMAILS_FROM_NAME", "My App")
```

#### Step 3: Create Email Utility
```python
# backend/app/core/email.py
from typing import List, Optional
from emails import Message
from jinja2 import Template
import aiosmtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

from app.core import config


async def send_email(
    email_to: str,
    subject: str,
    html_content: str,
) -> None:
    message = MIMEMultipart("alternative")
    message["From"] = f"{config.EMAILS_FROM_NAME} <{config.EMAILS_FROM_EMAIL}>"
    message["To"] = email_to
    message["Subject"] = subject

    html_part = MIMEText(html_content, "html")
    message.attach(html_part)

    await aiosmtplib.send(
        message,
        hostname=config.SMTP_HOST,
        port=config.SMTP_PORT,
        username=config.SMTP_USER,
        password=config.SMTP_PASSWORD,
        start_tls=True,
    )


def generate_welcome_email(user_email: str, user_name: str) -> str:
    html = f"""
    <html>
        <body>
            <h1>Welcome to {config.PROJECT_NAME}!</h1>
            <p>Hi {user_name},</p>
            <p>Thank you for signing up. We're excited to have you on board!</p>
        </body>
    </html>
    """
    return html
```

#### Step 4: Integrate with Signup
```python
# In backend/app/api/api_v1/routers/auth.py
from app.core.email import send_email, generate_welcome_email

@r.post("/signup")
async def signup(
    db=Depends(get_db), 
    form_data: OAuth2PasswordRequestForm = Depends(),
    background_tasks: BackgroundTasks  # Add this
):
    user = sign_up_new_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(...)
    
    # Send welcome email asynchronously
    background_tasks.add_task(
        send_email,
        email_to=user.email,
        subject="Welcome!",
        html_content=generate_welcome_email(user.email, user.first_name or "there")
    )
    
    # ... rest of the code
```

#### Step 5: Update Docker Compose
```yaml
# docker-compose.yml
# Add to backend service environment:
environment:
  SMTP_HOST: smtp.gmail.com
  SMTP_PORT: 587
  SMTP_USER: ${SMTP_USER}
  SMTP_PASSWORD: ${SMTP_PASSWORD}
  EMAILS_FROM_EMAIL: noreply@myapp.com
```

---

## 2. Password Reset Flow

### Backend Implementation

#### Step 1: Add Password Reset Token Model
```python
# backend/app/db/models.py
from datetime import datetime
import secrets

class PasswordResetToken(Base):
    __tablename__ = "password_reset_token"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("user.id"))
    token = Column(String, unique=True, index=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    expires_at = Column(DateTime)
    used = Column(Boolean, default=False)
    
    user = relationship("User", backref="reset_tokens")
```

#### Step 2: Create Migration
```bash
cd backend
alembic revision -m "add_password_reset_token"
# Edit the generated migration file
alembic upgrade head
```

#### Step 3: Add Reset Endpoints
```python
# backend/app/api/api_v1/routers/auth.py
from datetime import datetime, timedelta
import secrets

@r.post("/forgot-password")
async def forgot_password(
    email: str,
    db=Depends(get_db),
    background_tasks: BackgroundTasks
):
    user = db.query(models.User).filter(models.User.email == email).first()
    if not user:
        # Don't reveal if user exists
        return {"message": "If email exists, reset link will be sent"}
    
    # Create reset token
    token = secrets.token_urlsafe(32)
    reset_token = models.PasswordResetToken(
        user_id=user.id,
        token=token,
        expires_at=datetime.utcnow() + timedelta(hours=1)
    )
    db.add(reset_token)
    db.commit()
    
    # Send email
    reset_link = f"https://yourapp.com/reset-password?token={token}"
    background_tasks.add_task(
        send_email,
        email_to=user.email,
        subject="Password Reset Request",
        html_content=generate_reset_email(user.first_name, reset_link)
    )
    
    return {"message": "If email exists, reset link will be sent"}


@r.post("/reset-password")
async def reset_password(
    token: str,
    new_password: str,
    db=Depends(get_db)
):
    reset_token = db.query(models.PasswordResetToken).filter(
        models.PasswordResetToken.token == token,
        models.PasswordResetToken.used == False,
        models.PasswordResetToken.expires_at > datetime.utcnow()
    ).first()
    
    if not reset_token:
        raise HTTPException(status_code=400, detail="Invalid or expired token")
    
    # Update password
    user = reset_token.user
    user.hashed_password = security.get_password_hash(new_password)
    reset_token.used = True
    db.commit()
    
    return {"message": "Password updated successfully"}
```

### Frontend Implementation

#### Step 1: Create Forgot Password Page
```typescript
// frontend/src/views/ForgotPassword.tsx
import React, { useState } from 'react';
import { TextField, Button, Typography } from '@material-ui/core';
import { api } from '../utils/api';

export const ForgotPassword: React.FC = () => {
  const [email, setEmail] = useState('');
  const [message, setMessage] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await api.post('/api/forgot-password', { email });
      setMessage('If your email exists, you will receive a reset link');
    } catch (error) {
      setMessage('An error occurred. Please try again.');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <Typography variant="h5">Forgot Password</Typography>
      <TextField
        label="Email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        type="email"
        required
      />
      <Button type="submit">Send Reset Link</Button>
      {message && <Typography>{message}</Typography>}
    </form>
  );
};
```

#### Step 2: Create Reset Password Page
```typescript
// frontend/src/views/ResetPassword.tsx
import React, { useState } from 'react';
import { useHistory, useLocation } from 'react-router-dom';
import { TextField, Button, Typography } from '@material-ui/core';
import { api } from '../utils/api';

export const ResetPassword: React.FC = () => {
  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [message, setMessage] = useState('');
  const location = useLocation();
  const history = useHistory();
  
  const token = new URLSearchParams(location.search).get('token');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    if (password !== confirmPassword) {
      setMessage('Passwords do not match');
      return;
    }

    try {
      await api.post('/api/reset-password', { token, new_password: password });
      setMessage('Password reset successful! Redirecting to login...');
      setTimeout(() => history.push('/login'), 2000);
    } catch (error) {
      setMessage('Invalid or expired token');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <Typography variant="h5">Reset Password</Typography>
      <TextField
        label="New Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        type="password"
        required
      />
      <TextField
        label="Confirm Password"
        value={confirmPassword}
        onChange={(e) => setConfirmPassword(e.target.value)}
        type="password"
        required
      />
      <Button type="submit">Reset Password</Button>
      {message && <Typography>{message}</Typography>}
    </form>
  );
};
```

#### Step 3: Update Routes
```typescript
// frontend/src/Routes.tsx
import { ForgotPassword, ResetPassword } from './views';

// Add routes:
<Route path="/forgot-password" component={ForgotPassword} />
<Route path="/reset-password" component={ResetPassword} />
```

---

## 3. API Rate Limiting

### Backend Implementation

#### Step 1: Add Dependencies
```bash
# backend/requirements.txt
slowapi==0.1.8
```

#### Step 2: Configure Rate Limiter
```python
# backend/app/core/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address
from redis import Redis

redis_client = Redis(host='redis', port=6379, db=1, decode_responses=True)

limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://redis:6379/1",
    default_limits=["100/hour"]
)
```

#### Step 3: Apply to FastAPI App
```python
# backend/app/main.py
from slowapi import _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from app.core.rate_limit import limiter

app = FastAPI(...)

app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

#### Step 4: Apply to Login Endpoint
```python
# backend/app/api/api_v1/routers/auth.py
from app.core.rate_limit import limiter
from fastapi import Request

@r.post("/token")
@limiter.limit("5/minute")  # 5 login attempts per minute
async def login(
    request: Request,  # Required for rate limiter
    db=Depends(get_db),
    form_data: OAuth2PasswordRequestForm = Depends()
):
    # ... existing code
```

#### Step 5: Apply to User Endpoints
```python
# backend/app/api/api_v1/routers/users.py
from app.core.rate_limit import limiter

@r.get("/users")
@limiter.limit("30/minute")
async def users_list(...):
    # ... existing code

@r.post("/users")
@limiter.limit("10/minute")
async def user_create(...):
    # ... existing code
```

---

## 4. Comprehensive Logging

### Backend Implementation

#### Step 1: Add Dependencies
```bash
# backend/requirements.txt
loguru==0.6.0
python-json-logger==2.0.4
```

#### Step 2: Configure Logging
```python
# backend/app/core/logging_config.py
import sys
from loguru import logger
from pathlib import Path

# Remove default logger
logger.remove()

# Add console logger with colors
logger.add(
    sys.stdout,
    colorize=True,
    format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>",
    level="INFO"
)

# Add file logger
log_path = Path("logs")
log_path.mkdir(exist_ok=True)

logger.add(
    "logs/app_{time:YYYY-MM-DD}.log",
    rotation="00:00",  # New file at midnight
    retention="30 days",  # Keep logs for 30 days
    compression="zip",
    level="DEBUG",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {name}:{function}:{line} - {message}"
)

# Add error-only logger
logger.add(
    "logs/error_{time:YYYY-MM-DD}.log",
    rotation="00:00",
    retention="90 days",
    level="ERROR",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {name}:{function}:{line} - {message}\n{exception}"
)
```

#### Step 3: Add Request Logging Middleware
```python
# backend/app/main.py
from app.core.logging_config import logger
import time

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    
    logger.info(f"Request: {request.method} {request.url.path}")
    
    try:
        response = await call_next(request)
        process_time = time.time() - start_time
        
        logger.info(
            f"Response: {request.method} {request.url.path} "
            f"Status: {response.status_code} Time: {process_time:.3f}s"
        )
        
        return response
    except Exception as e:
        logger.error(f"Request failed: {request.method} {request.url.path} Error: {str(e)}")
        raise
```

#### Step 4: Use in Endpoints
```python
# In any endpoint
from app.core.logging_config import logger

@r.post("/users")
async def user_create(...):
    logger.info(f"Creating new user: {user.email}")
    try:
        created_user = create_user(db, user)
        logger.info(f"User created successfully: {created_user.id}")
        return created_user
    except Exception as e:
        logger.error(f"Failed to create user {user.email}: {str(e)}")
        raise
```

---

## 5. User Profile Management

### Backend Implementation

#### Step 1: Add Profile Update Endpoint
```python
# backend/app/api/api_v1/routers/users.py
from app.db.schemas import UserProfileUpdate

@r.put("/users/me", response_model=User)
async def update_own_profile(
    profile: UserProfileUpdate,
    db=Depends(get_db),
    current_user=Depends(get_current_active_user),
):
    """
    Update own user profile
    """
    # Users can only update their own profile
    return edit_user(db, current_user.id, profile)


@r.delete("/users/me")
async def delete_own_account(
    password: str,
    db=Depends(get_db),
    current_user=Depends(get_current_active_user),
):
    """
    Delete own account (GDPR compliance)
    """
    # Verify password before deletion
    if not security.verify_password(password, current_user.hashed_password):
        raise HTTPException(status_code=400, detail="Incorrect password")
    
    # Soft delete or hard delete based on requirements
    current_user.is_active = False
    db.commit()
    
    return {"message": "Account deleted successfully"}
```

#### Step 2: Add Schema
```python
# backend/app/db/schemas.py
class UserProfileUpdate(BaseModel):
    first_name: Optional[str] = None
    last_name: Optional[str] = None
    # Add more profile fields as needed
    
    class Config:
        orm_mode = True
```

### Frontend Implementation

```typescript
// frontend/src/views/Profile.tsx
import React, { useState, useEffect } from 'react';
import { TextField, Button, Typography } from '@material-ui/core';
import { api } from '../utils/api';

export const Profile: React.FC = () => {
  const [profile, setProfile] = useState({
    email: '',
    first_name: '',
    last_name: '',
  });

  useEffect(() => {
    // Load current user profile
    api.get('/api/v1/users/me').then(response => {
      setProfile(response.data);
    });
  }, []);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await api.put('/api/v1/users/me', profile);
      alert('Profile updated successfully');
    } catch (error) {
      alert('Failed to update profile');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <Typography variant="h5">My Profile</Typography>
      <TextField
        label="Email"
        value={profile.email}
        disabled
      />
      <TextField
        label="First Name"
        value={profile.first_name}
        onChange={(e) => setProfile({...profile, first_name: e.target.value})}
      />
      <TextField
        label="Last Name"
        value={profile.last_name}
        onChange={(e) => setProfile({...profile, last_name: e.target.value})}
      />
      <Button type="submit">Update Profile</Button>
    </form>
  );
};
```

---

## 6. File Upload Support

### Backend Implementation

#### Step 1: Add File Model
```python
# backend/app/db/models.py
class UploadedFile(Base):
    __tablename__ = "uploaded_file"
    
    id = Column(Integer, primary_key=True, index=True)
    filename = Column(String)
    original_filename = Column(String)
    content_type = Column(String)
    size = Column(Integer)
    user_id = Column(Integer, ForeignKey("user.id"))
    uploaded_at = Column(DateTime, default=datetime.utcnow)
    file_path = Column(String)
    
    user = relationship("User", backref="files")
```

#### Step 2: Create Upload Endpoint
```python
# backend/app/api/api_v1/routers/files.py
from fastapi import APIRouter, File, UploadFile, Depends
import shutil
from pathlib import Path
import uuid

files_router = r = APIRouter()

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)

@r.post("/upload")
async def upload_file(
    file: UploadFile = File(...),
    db=Depends(get_db),
    current_user=Depends(get_current_active_user)
):
    # Validate file type
    allowed_types = ["image/jpeg", "image/png", "image/gif", "application/pdf"]
    if file.content_type not in allowed_types:
        raise HTTPException(status_code=400, detail="File type not allowed")
    
    # Validate file size (e.g., 5MB)
    max_size = 5 * 1024 * 1024
    file.file.seek(0, 2)
    file_size = file.file.tell()
    file.file.seek(0)
    
    if file_size > max_size:
        raise HTTPException(status_code=400, detail="File too large")
    
    # Generate unique filename
    file_extension = Path(file.filename).suffix
    unique_filename = f"{uuid.uuid4()}{file_extension}"
    file_path = UPLOAD_DIR / unique_filename
    
    # Save file
    with file_path.open("wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    
    # Save to database
    db_file = models.UploadedFile(
        filename=unique_filename,
        original_filename=file.filename,
        content_type=file.content_type,
        size=file_size,
        user_id=current_user.id,
        file_path=str(file_path)
    )
    db.add(db_file)
    db.commit()
    
    return {
        "id": db_file.id,
        "filename": unique_filename,
        "size": file_size
    }

@r.get("/files/{file_id}")
async def download_file(
    file_id: int,
    db=Depends(get_db),
    current_user=Depends(get_current_active_user)
):
    from fastapi.responses import FileResponse
    
    db_file = db.query(models.UploadedFile).filter(
        models.UploadedFile.id == file_id
    ).first()
    
    if not db_file:
        raise HTTPException(status_code=404, detail="File not found")
    
    # Check permissions
    if db_file.user_id != current_user.id and not current_user.is_superuser:
        raise HTTPException(status_code=403, detail="Not authorized")
    
    return FileResponse(
        db_file.file_path,
        filename=db_file.original_filename,
        media_type=db_file.content_type
    )
```

### Frontend Implementation

```typescript
// frontend/src/components/FileUpload.tsx
import React, { useState } from 'react';
import { Button, Typography, LinearProgress } from '@material-ui/core';
import { api } from '../utils/api';

export const FileUpload: React.FC = () => {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);

  const handleFileSelect = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    const formData = new FormData();
    formData.append('file', file);

    setUploading(true);
    try {
      const response = await api.post('/api/v1/upload', formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
        onUploadProgress: (progressEvent) => {
          const percentCompleted = Math.round(
            (progressEvent.loaded * 100) / progressEvent.total
          );
          setProgress(percentCompleted);
        },
      });

      alert(`File uploaded successfully! ID: ${response.data.id}`);
    } catch (error) {
      alert('Upload failed');
    } finally {
      setUploading(false);
      setProgress(0);
    }
  };

  return (
    <div>
      <input
        accept="image/*,.pdf"
        style={{ display: 'none' }}
        id="file-upload"
        type="file"
        onChange={handleFileSelect}
        disabled={uploading}
      />
      <label htmlFor="file-upload">
        <Button variant="contained" component="span" disabled={uploading}>
          {uploading ? 'Uploading...' : 'Upload File'}
        </Button>
      </label>
      {uploading && (
        <>
          <LinearProgress variant="determinate" value={progress} />
          <Typography>{progress}%</Typography>
        </>
      )}
    </div>
  );
};
```

---

## Testing Your Implementations

### Email Testing (Development)
1. Use MailHog for local testing:
```yaml
# Add to docker-compose.yml
mailhog:
  image: mailhog/mailhog
  ports:
    - "1025:1025"  # SMTP
    - "8025:8025"  # Web UI
```

2. Configure backend to use MailHog:
```
SMTP_HOST=mailhog
SMTP_PORT=1025
```

3. View emails at http://localhost:8025

### Rate Limiting Testing
```bash
# Test with curl
for i in {1..10}; do curl -X POST http://localhost:8000/api/token -d "username=test&password=test"; done
```

### File Upload Testing
```bash
curl -X POST http://localhost:8000/api/v1/upload \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "file=@test.jpg"
```

---

## Deployment Considerations

### Environment Variables
Create a `.env.production` file:
```env
# Email
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASSWORD=your_sendgrid_api_key
EMAILS_FROM_EMAIL=noreply@yourapp.com

# Security
SECRET_KEY=your_super_secret_key_here

# Database
DATABASE_URL=postgresql://user:pass@host:5432/db

# Redis
REDIS_URL=redis://redis:6379
```

### Docker Volumes
```yaml
volumes:
  - ./uploads:/app/uploads  # For file uploads
  - ./logs:/app/logs        # For log files
```

### Nginx Configuration
```nginx
# Increase upload size limit
client_max_body_size 10M;

# Add rate limiting at nginx level
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
location /api/token {
    limit_req zone=login burst=3 nodelay;
}
```

---

## Next Steps After Implementation

1. **Test Thoroughly**
   - Unit tests for new endpoints
   - Integration tests for workflows
   - Load testing for rate limiting

2. **Update Documentation**
   - API documentation with examples
   - User guide for new features
   - Admin guide for email configuration

3. **Monitor**
   - Email delivery rates
   - Rate limit hit rates
   - File storage usage
   - Log aggregation

4. **Security Audit**
   - Penetration testing
   - Code review
   - Dependency scanning

---

*This implementation guide provides practical, production-ready code for the top priority features. Adjust configurations based on your specific needs.*
