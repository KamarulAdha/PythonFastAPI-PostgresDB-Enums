# Enum Implementation Flashcards: FastAPI, PostgreSQL, and SQLAlchemy

## Core Concepts

**What is an enum?**  
A type that restricts column values to a predefined set of options.

**What are the three implementation approaches for enums?**  
1. Application-Level Validation (Simple)
2. String with Check Constraint (Moderate)
3. Native PostgreSQL Enums (Complex)

## Approach 1: Application-Level Validation

**Implementation**  
Database: Regular VARCHAR/TEXT column  
SQLAlchemy: Standard String column type  
Validation: Handled in application code  

**Database Definition**  
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    user_type VARCHAR NOT NULL
);
```

**SQLAlchemy Model**  
```python
class UserType(enum.Enum):
    CUSTOMER = "customer"
    OWNER = "owner"
    ADMIN = "admin"

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    user_type = Column(String, nullable=False)
```

**Key Advantages**  
- Simplest implementation
- No database migrations for enum changes
- Maximum flexibility

**Key Disadvantages**  
- No database-level validation
- Risk of inconsistency if data inserted through other means

**Best Used When**  
- Early development with changing requirements
- Single entry point for all data
- Need to avoid database migrations

## Approach 2: String with Check Constraint

**Implementation**  
Database: String column with CHECK constraint  
SQLAlchemy: Define enum in Python and CHECK constraint  
Validation: At both application and database levels  

**SQLAlchemy Model**  
```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    user_type = Column(
        String, 
        CheckConstraint("user_type IN ('customer', 'owner', 'admin')"),
        nullable=False
    )
```

**Alternative SQLAlchemy Syntax**  
```python
user_type = Column(
    Enum(UserType, native_enum=False),
    nullable=False
)
```

**Migration for Adding Values**  
```python
def upgrade():
    op.drop_constraint('users_user_type_check', 'users')
    op.create_check_constraint(
        'users_user_type_check', 
        'users', 
        "user_type IN ('customer', 'owner', 'admin', 'manager')"
    )
```

**Key Advantages**  
- Database-level validation
- Easier to modify than native enums
- Validation at both database and API levels

**Key Disadvantages**  
- Requires synchronized changes in code and database
- Need for migrations with any enum value change

**Best Used When**  
- Semi-stable categories that change occasionally
- Database validation is important but flexibility needed
- Frequent iterative development

## Approach 3: Native PostgreSQL Enums

**Implementation**  
Database: Custom ENUM type and column using that type  
SQLAlchemy: Define enum in Python and use SQLAlchemy Enum type  
Validation: At all levels - database, ORM, and API  

**SQLAlchemy Model**  
```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    user_type = Column(Enum(UserType), nullable=False)
```

**Resulting PostgreSQL Schema**  
```sql
CREATE TYPE user_type AS ENUM ('customer', 'owner', 'admin');

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    user_type user_type NOT NULL
);
```

**Migration for Adding Values**  
```python
def upgrade():
    op.execute("ALTER TYPE user_type ADD VALUE 'manager'")
```

**Key Advantages**  
- Strong database-level type safety
- Self-documenting schema
- Compact storage (more efficient than strings)

**Key Disadvantages**  
- Most complex to modify (especially removing values)
- Requires careful coordination between code and database
- Complex migrations required for enum changes

**Best Used When**  
- Fixed category lists that rarely change
- Database-level type safety is high priority
- Regulated environments requiring data integrity

## Integration with FastAPI/Pydantic

**Pydantic Validation (Approach 1)**  
```python
class UserCreate(BaseModel):
    user_type: UserType  # Validates against the enum
    
    class Config:
        orm_mode = True

@app.post("/users/")
async def create_user(user: UserCreate):
    db_user = User(
        user_type=user.user_type.value  # Store string value
    )
```

**Manual Validation (Approach 1)**  
```python
@app.post("/users/")
async def create_user(user: UserCreate):
    valid_types = [type.value for type in UserType]
    if user.user_type not in valid_types:
        raise HTTPException(
            status_code=400,
            detail=f"Invalid user_type. Must be one of: {', '.join(valid_types)}"
        )
```

**Pydantic with Native Enum (Approach 3)**  
```python
class UserCreate(BaseModel):
    user_type: UserType  # Uses the same enum
    
    class Config:
        orm_mode = True

@app.post("/users/")
async def create_user(user: UserCreate):
    db_user = User(
        user_type=user.user_type  # Direct assignment works
    )
```

## Quick Reference

**Complexity Comparison**  
- Application-Level: ⭐ Simple (code only)
- Check Constraint: ⭐⭐ Moderate (code + DB constraint)
- Native PostgreSQL: ⭐⭐⭐ Complex (code + DB type + complex migrations)

**When to Use Each Approach**  
- Application-Level: Early development, rapid changes
- Check Constraint: Semi-stable categories, need both flexibility and validation
- Native PostgreSQL: Stable categories, high data integrity requirements

**Real-World Examples**  
- Application-Level: User roles in early-stage apps
- Check Constraint: Order statuses in e-commerce
- Native PostgreSQL: Payment methods in financial applications