# Using Enums with FastAPI, PostgreSQL, and SQLAlchemy

## Introduction

Enums (enumerated types) provide a way to restrict column values to a predefined set of options. When working with FastAPI, PostgreSQL, and SQLAlchemy, there are several approaches to implementing enums, each with different tradeoffs.

This guide explores the various implementation options, from the simplest to most complex, providing clear examples and practical guidance on when to use each approach.

## Implementation Approaches

### 1. Application-Level Validation

**Complexity Level: Simple** ⭐
**Changes Required:** Backend code only (no database schema changes)

This approach uses a plain string column in the database but relies on your application code for validation. It's the simplest to implement and offers the most flexibility for changes.

| Component | Changes Required |
|-----------|------------------|
| Database | None (just a regular VARCHAR/TEXT column) |
| SQLAlchemy Model | Use standard String column type |
| Validation | Handled in application code |
| Migrations | Not needed for enum value changes |

#### Database Level
Just a simple string column:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    user_type VARCHAR NOT NULL
    -- other columns
);
```

#### SQLAlchemy Model

```python
import enum
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

# Define the enum for use in validation
class UserType(enum.Enum):
    CUSTOMER = "customer"
    OWNER = "owner"
    ADMIN = "admin"

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    user_type = Column(String, nullable=False)  # Just a regular string column
    # other columns...
```

#### Adding/Changing Enum Values

To add or change values, simply update the Python enum definition. No database changes needed:

```python
class UserType(enum.Enum):
    CUSTOMER = "customer"
    OWNER = "owner"
    ADMIN = "admin"
    MANAGER = "manager"  # New value added
```

There are two common variations for implementing validation with this approach:

#### Variation A: Pydantic Validation (Recommended)

This variation leverages FastAPI's built-in Pydantic validation, making it both simple and elegant:

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel
from sqlalchemy.orm import Session

app = FastAPI()

# Pydantic model with enum validation
class UserCreate(BaseModel):
    username: str
    user_type: UserType  # Validates against the enum
    # other fields...

    class Config:
        orm_mode = True

# FastAPI endpoint
@app.post("/users/")
async def create_user(user: UserCreate, db: Session = Depends(get_db)):
    # FastAPI/Pydantic automatically validates that user_type is a valid enum value
    # Invalid values are rejected with a 422 Unprocessable Entity error
    
    db_user = User(
        username=user.username,
        user_type=user.user_type.value  # Store the string value in the database
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user
```

**Advantages:**
- Minimal code required
- Automatic validation during request parsing
- Standard error responses
- Self-documenting API schema

#### Variation B: Manual Validation

This variation uses custom validation code, giving you more control over the validation process:

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from sqlalchemy.orm import Session

app = FastAPI()

class UserCreate(BaseModel):
    username: str
    user_type: str  # String, not enum - no automatic validation
    # other fields...

    class Config:
        orm_mode = True

@app.post("/users/")
async def create_user(user: UserCreate, db: Session = Depends(get_db)):
    # Manual validation
    valid_types = [type.value for type in UserType]
    if user.user_type not in valid_types:
        raise HTTPException(
            status_code=400,
            detail=f"Invalid user_type. Must be one of: {', '.join(valid_types)}"
        )
    
    db_user = User(
        username=user.username,
        user_type=user.user_type
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user
```

**Advantages:**
- Complete control over validation logic and error messages
- Can implement complex validation rules beyond simple enum checking
- Flexibility to handle special cases or transitions between values

### 2. String with Check Constraint

**Complexity Level: Moderate** ⭐⭐
**Changes Required:** Backend code and database constraint

This approach uses a regular string column but adds a database constraint to enforce valid values, providing validation at both the application and database levels.

| Component | Changes Required |
|-----------|------------------|
| Database | String column with CHECK constraint |
| SQLAlchemy Model | Define enum in Python and CHECK constraint |
| Pydantic Schema | Use Python Enum for the field |
| Endpoint | No special handling required |
| Migrations | Need to update constraint when enum values change |

#### Python/SQLAlchemy Code

```python
import enum
from sqlalchemy import Column, Integer, String, CheckConstraint
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class UserType(enum.Enum):
    CUSTOMER = "customer"
    OWNER = "owner"
    ADMIN = "admin"

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    user_type = Column(
        String, 
        CheckConstraint("user_type IN ('customer', 'owner', 'admin')"),
        nullable=False
    )
    # other columns...
```

#### Alternative: Using SQLAlchemy's Enum with native_enum=False

```python
user_type = Column(
    Enum(UserType, native_enum=False),
    nullable=False
)
```

This produces the same result: a VARCHAR with a CHECK constraint.

#### Changing Values with Check Constraints

You need to update both the Python enum and the database constraint:

```python
# In your Alembic migration
from alembic import op

def upgrade():
    # Update the check constraint
    op.drop_constraint('users_user_type_check', 'users')
    op.create_check_constraint(
        'users_user_type_check', 
        'users', 
        "user_type IN ('customer', 'owner', 'admin', 'manager')"
    )

def downgrade():
    op.drop_constraint('users_user_type_check', 'users')
    op.create_check_constraint(
        'users_user_type_check', 
        'users', 
        "user_type IN ('customer', 'owner', 'admin')"
    )
```

### 3. Native PostgreSQL Enums

**Complexity Level: Complex** ⭐⭐⭐
**Changes Required:** Backend code, database type definition, and migrations

This approach defines an enum type at both the Python and database levels. PostgreSQL creates a custom enum type in the database that only accepts the specified values, providing the strongest validation but with the highest complexity.

| Component | Changes Required |
|-----------|------------------|
| Database | Custom ENUM type and column using that type |
| SQLAlchemy Model | Define enum in Python and use SQLAlchemy Enum type |
| Pydantic Schema | Use Python Enum for the field |
| Endpoint | No special handling required |
| Migrations | Complex migrations for enum value changes |

#### Python/SQLAlchemy Code

```python
import enum
from sqlalchemy import Column, Integer, Enum
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class UserType(enum.Enum):
    CUSTOMER = "customer"
    OWNER = "owner"
    ADMIN = "admin"

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    user_type = Column(Enum(UserType), nullable=False)  # Uses native PostgreSQL enum
    # other columns...
```

#### Resulting PostgreSQL Schema

```sql
-- This gets created in PostgreSQL
CREATE TYPE user_type AS ENUM ('customer', 'owner', 'admin');

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    user_type user_type NOT NULL
    -- other columns
);
```

#### FastAPI/Pydantic Integration

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class UserCreate(BaseModel):
    username: str
    user_type: UserType  # Uses the same enum
    # other fields...

    class Config:
        orm_mode = True

@app.post("/users/")
async def create_user(user: UserCreate):
    # FastAPI automatically validates that user_type is a valid enum value
    # Invalid values are rejected with a 422 Unprocessable Entity error
    db_user = User(
        username=user.username,
        user_type=user.user_type
    )
    # Add to DB, commit, etc.
    return db_user
```

#### Adding Values to Native PostgreSQL Enums

1. Update Python Code:

```python
class UserType(enum.Enum):
    CUSTOMER = "customer"
    OWNER = "owner"
    ADMIN = "admin"
    MANAGER = "manager"  # New value
```

2. Update Database Schema using Alembic:

```python
# In your Alembic migration
from alembic import op

def upgrade():
    op.execute("ALTER TYPE user_type ADD VALUE 'manager'")

def downgrade():
    # Cannot easily remove values - usually left empty
    pass
```

#### Removing or Reordering PostgreSQL Enum Values

This is the most complex scenario and requires recreating the type:

```python
# In your Alembic migration
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import ENUM

def upgrade():
    # Create new enum type
    op.execute("CREATE TYPE user_type_new AS ENUM ('customer', 'manager', 'admin')")
    
    # Create temporary column with new type
    op.add_column('users', sa.Column('user_type_new', 
                  ENUM('customer', 'manager', 'admin', name='user_type_new'),
                  nullable=True))
    
    # Update data (with mapping for removed values)
    op.execute("""
    UPDATE users 
    SET user_type_new = CASE
        WHEN user_type = 'owner' THEN 'manager'
        ELSE user_type::text::user_type_new
    END
    """)
    
    # Drop old column and rename new one
    op.drop_column('users', 'user_type')
    op.alter_column('users', 'user_type_new', new_column_name='user_type', nullable=False)
    
    # Drop old enum type
    op.execute("DROP TYPE user_type")
    
    # Rename new enum type to match old name
    op.execute("ALTER TYPE user_type_new RENAME TO user_type")

def downgrade():
    # Similar complex reversal process
    pass
```

## Implementation Complexity Comparison

| Approach | Database Changes | Model Changes | Schema/Validation | Migration Complexity | Overall Complexity |
|----------|------------------|---------------|-------------------|---------------------|-------------------|
| Application-Level (Pydantic) | None | Simple | Simple | None | ⭐ Simple |
| Application-Level (Manual) | None | Simple | Moderate | None | ⭐ Simple |
| Check Constraint | Moderate | Moderate | Simple | Moderate | ⭐⭐ Moderate |
| Native PostgreSQL Enum | Complex | Moderate | Simple | Complex | ⭐⭐⭐ Complex |

## Pros and Cons

### Application-Level Validation

**Pros:**
- Simplest implementation - just a string in the database
- No database migrations needed when enum values change
- Maximum flexibility for development and changes
- When using Pydantic: clean, declarative validation
- When using manual validation: complete control over validation logic

**Cons:**
- No database-level protection against invalid values
- Depends entirely on application validation
- Risk of inconsistency if data is inserted through other means
- Database doesn't self-document valid values

### String with Check Constraint

**Pros:**
- Provides database-level validation
- Easier to modify than native enums
- Visible constraints in database tools
- Validation at both database and API levels

**Cons:**
- Requires synchronized changes in code and database
- Need for migrations with any enum value change
- Constraint definition might get out of sync with code

### Native PostgreSQL Enums

**Pros:**
- Strong database-level type safety
- Self-documenting schema
- Compact storage (more efficient than strings)
- Validation at all levels: database, ORM, and API

**Cons:**
- Most complex to modify (especially removing or reordering values)
- Requires careful coordination between code and database
- Complex migrations required for enum changes

## Use Case Recommendations

### When to Use Application-Level Validation

- During early development with rapidly changing requirements
- When there's a single entry point (API) for all data
- When you want to minimize complexity and maximize flexibility
- When you need to avoid database migrations

**Pydantic Variation:** Use when you want clean, standardized validation with minimal code.

**Manual Validation Variation:** Use when you need complex validation rules or special handling of certain values.

**Example**: A startup building an MVP where features and data models change frequently.

### When to Use Check Constraints

- Semi-stable categories that change occasionally
- When you want database validation with more flexibility than native enums
- In applications with frequent iterative development but database validation is important

**Example**: Product categories in an e-commerce application that might have new types added every few months.

### When to Use Native PostgreSQL Enums

- Fixed category lists that rarely change (e.g., user roles, subscription tiers)
- When database-level type safety is a high priority
- In regulated environments where data integrity must be enforced at all levels
- When you have solid migration practices in place

**Example**: A banking application with transaction types like 'deposit', 'withdrawal', 'transfer' - these rarely change.

## Real-World Implementation Examples

### Example 1: User Roles in an Early-Stage Application

For a startup developing a new product with evolving user roles, Application-Level Validation provides the flexibility needed during early development:

```python
# models.py
import enum
from sqlalchemy import Column, Integer, String

class UserRole(enum.Enum):
    USER = "user"
    ADMIN = "admin"
    BETA_TESTER = "beta_tester"  # Easy to add new roles

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    role = Column(String, nullable=False)

# schemas.py
from pydantic import BaseModel

class UserCreate(BaseModel):
    username: str
    role: UserRole  # Validated by Pydantic
```

### Example 2: Order Status with Database Validation

For an e-commerce platform where order status is critical, Check Constraints provide database-level validation while maintaining reasonable flexibility:

```python
# models.py
import enum
from sqlalchemy import Column, Integer, String, CheckConstraint

class OrderStatus(enum.Enum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    status = Column(
        String,
        CheckConstraint("status IN ('pending', 'paid', 'shipped', 'delivered', 'cancelled')"),
        nullable=False
    )
```

### Example 3: Payment Methods in a Financial Application

For a financial application where payment methods are standardized and change infrequently, Native PostgreSQL Enums provide the strongest type safety:

```python
# models.py
import enum
from sqlalchemy import Column, Integer, Enum

class PaymentMethod(enum.Enum):
    CREDIT_CARD = "credit_card"
    DEBIT_CARD = "debit_card"
    BANK_TRANSFER = "bank_transfer"
    DIGITAL_WALLET = "digital_wallet"

class Payment(Base):
    __tablename__ = "payments"
    id = Column(Integer, primary_key=True)
    method = Column(Enum(PaymentMethod), nullable=False)  # Native PostgreSQL enum
```

## Summary

1. **Application-Level Validation** (⭐): The simplest approach with two variations:
   - **Pydantic Validation**: Clean, declarative validation with automatic error handling
   - **Manual Validation**: Custom validation code with more control over the process

   Both variations require changes only to backend code, with no database modifications. This approach offers maximum flexibility for development but no database-level protection.

2. **Check Constraints** (⭐⭐): Moderate complexity requiring changes to both backend code and database constraints. Offers better database-level validation while maintaining reasonable flexibility.

3. **Native PostgreSQL Enums** (⭐⭐⭐): Highest complexity requiring changes to backend code, database type definitions, and complex migrations. Provides the strongest type safety but is the most difficult to modify.

Choose the approach that aligns with your project's stability needs, development pace, and data integrity requirements. Many projects begin with Application-Level Validation during early development and migrate to more restrictive approaches as requirements stabilize.

The key insight is that validation can happen at multiple levels in your application stack. The simplest approach provides validation only at the application level, while more complex approaches add database-level validation for stronger data integrity guarantees.

Remember that consistency between your code and database is essential for all approaches - whether enforced by the database or maintained through your application code.