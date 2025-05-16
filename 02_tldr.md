# TLDR: Enums with FastAPI and PostgreSQL

## Implementation Approaches at a Glance

| Approach | Complexity | Database Schema | Python/FastAPI | Changes Required | Migration Complexity |
|----------|------------|-----------------|----------------|------------------|---------------------|
| **1. Application-Level** | ⭐ Simple | Regular VARCHAR/TEXT | Python enum + validation | Backend code only | None |
| **2. Check Constraint** | ⭐⭐ Moderate | VARCHAR + CHECK constraint | Python enum + validation | Code + DB constraint | Moderate |
| **3. Native PostgreSQL Enum** | ⭐⭐⭐ Complex | Custom ENUM type | Python enum + SQLAlchemy Enum type | Code + DB type + migrations | Complex |

## Pros and Cons

### 1. Application-Level Validation

**Pros:**
- Simplest implementation - just strings in database
- No database migrations when enum values change
- Maximum flexibility during development
- Clean validation with Pydantic integration

**Cons:**
- No database-level protection
- Risk of data inconsistency if bypassing API
- Database doesn't self-document valid values

### 2. String with Check Constraint

**Pros:**
- Database-level validation
- Easier to modify than native enums
- Self-documenting database constraints
- Validation at both database and API levels

**Cons:**
- Requires coordinated changes in code and database
- Needs migrations for any enum value change
- Constraints can get out of sync with code

### 3. Native PostgreSQL Enum

**Pros:**
- Strong database-level type safety
- Self-documenting schema
- Storage efficiency
- Validation at all levels

**Cons:**
- Most complex to modify (especially removing values)
- Complex migrations required for changes
- Requires careful coordination between code and database

## When to Use Each Approach

- **Application-Level:** Early development, rapidly changing requirements, MVP development
- **Check Constraint:** Semi-stable categories, iterative development with DB validation needs
- **Native PostgreSQL:** Stable, rarely changing categories, highly regulated environments

## Quick Implementation Reference

### 1. Application-Level (Python-only, DB stores strings)

```python
# Python enum
class UserType(enum.Enum):
    CUSTOMER = "customer"
    ADMIN = "admin"

# SQLAlchemy model
class User(Base):
    __tablename__ = "users"
    user_type = Column(String, nullable=False)  # Simple string

# Pydantic validation
class UserCreate(BaseModel):
    user_type: UserType  # Validates against enum
    
# FastAPI endpoint
@app.post("/users/")
async def create_user(user: UserCreate):
    db_user = User(user_type=user.user_type.value)  # Store string value
```

### 2. Check Constraint

```python
# SQLAlchemy model with check constraint
class User(Base):
    __tablename__ = "users"
    user_type = Column(
        String, 
        CheckConstraint("user_type IN ('customer', 'admin')"),
        nullable=False
    )

# Alternative syntax
user_type = Column(Enum(UserType, native_enum=False), nullable=False)  # Creates VARCHAR + CHECK
```

### 3. Native PostgreSQL Enum

```python
# SQLAlchemy model with native enum
class User(Base):
    __tablename__ = "users"
    user_type = Column(Enum(UserType), nullable=False)  # Creates custom enum type
```

## Common Pitfalls

- **Inconsistency between code and database** - Always ensure enum values match between Python and DB
- **Complex migrations** - Adding values is easy, but removing/renaming requires complex migrations
- **Forgotten constraints** - Check constraints must be manually updated in migrations
- **Validation gaps** - Application-level validation can be bypassed if data is inserted through other means
- **Over-engineering** - Using complex PostgreSQL enums when requirements are rapidly changing