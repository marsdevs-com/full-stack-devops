# Backend API: CRUD Example (Skills)

**Base path:** `/api/v1/skills`

---

## API

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/skills/` | List all skills (ordered by category, name). |
| POST | `/api/v1/skills/` | Create a skill. |
| PUT | `/api/v1/skills/{skill_id}` | Update a skill. |
| DELETE | `/api/v1/skills/{skill_id}` | Delete a skill (blocked if used by jobseekers or job postings). |

**Response:** All endpoints return the project's standard `APIResponse`:

```json
{
  "status": 200,
  "message": "Skills retrieved successfully.",
  "data": [...],
  "error": ""
}
```

---

## Model (DB layer)


```python
from sqlalchemy import Column, Integer, String, Table, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import UUID
import uuid

from app.db.database import Base

# Association table for JobseekerProfile and Skill
jobseeker_skills = Table(
    "jobseeker_skills",
    Base.metadata,
    Column("jobseeker_profile_id", UUID(as_uuid=True), ForeignKey("jobseeker_profiles.id", ondelete="CASCADE"), primary_key=True),
    Column("skill_id", UUID(as_uuid=True), ForeignKey("skills.id", ondelete="CASCADE"), primary_key=True),
)

# Association table for JobPosting and Skill
job_posting_skills = Table(
    "job_posting_skills",
    Base.metadata,
    Column("job_posting_id", UUID(as_uuid=True), ForeignKey("job_postings.id", ondelete="CASCADE"), primary_key=True),
    Column("skill_id", UUID(as_uuid=True), ForeignKey("skills.id", ondelete="CASCADE"), primary_key=True),
)

class Skill(Base):
    __tablename__ = "skills"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4, index=True)
    name = Column(String, unique=True, index=True, nullable=False)
    category = Column(String, index=True, nullable=True)  # e.g., 'Programming Language', 'Framework', 'Soft Skill'

    # Relationships to JobseekerProfile and JobPosting are defined in those models
    jobseekers = relationship("JobseekerProfile", secondary=jobseeker_skills, back_populates="skills")
    job_postings = relationship("JobPosting", secondary=job_posting_skills, back_populates="skills")
```

---

## Schema (request/response)


```python
# Base fields shared by Create, Update, and response
class SkillBase(BaseModel):
    name: str
    category: Optional[str] = None  # e.g. "Programming Language", "Framework", "Soft Skill"

# Request body for PUT /api/v1/skills/{skill_id} — all fields optional for partial updates
class SkillUpdate(SkillBase):
    name: Optional[str] = None
    category: Optional[str] = None

# Response/serialization schema; includes id from DB. Config allows building from ORM (Skill model)
class Skill(SkillBase):
    id: UUID

    class Config:
        from_attributes = True  # Pydantic v2: allow construction from SQLAlchemy model instances
```

---

## API (Router)

```python
from typing import List
from fastapi import APIRouter, Depends, HTTPException, status
from uuid import UUID

from app.api import deps
from app.db.models.user import User
from app.schemas.skill import Skill, SkillCreate, SkillUpdate
from app.services.skill import SkillService, get_skill_service
from app.utils.response_format import APIResponse

router = APIRouter()

@router.get("/", response_model=APIResponse)
def get_all_skills(
    skill_service: SkillService = Depends(get_skill_service),
) -> APIResponse:
    """
    Get all available skills with their IDs and categories.
    Open to all users.
    """
    skills = skill_service.get_all_skills()
    return APIResponse(
        status=200,
        message="Skills retrieved successfully.",
        data=[
            {"id": str(skill.id), "name": skill.name, "category": skill.category}
            for skill in skills
        ],
    )

@router.post("/", response_model=APIResponse, status_code=status.HTTP_201_CREATED)
def create_skill(
    *,
    skill_in: SkillCreate,
    skill_service: SkillService = Depends(get_skill_service),
    current_user: User = Depends(deps.required_admin_only),
) -> APIResponse:
    """
    Create a new skill. Admin access only.
    """
    # Check if skill with same name already exists
    existing_skill = skill_service.db.query(skill_service.model).filter(
        skill_service.model.name.ilike(skill_in.name)
    ).first()
    if existing_skill:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="A skill with this name already exists",
        )
    skill = skill_service.create(db=skill_service.db, obj_in=skill_in)
    return APIResponse(
        status=status.HTTP_201_CREATED,
        message="Skill created successfully.",
        data={"id": str(skill.id), "name": skill.name, "category": skill.category},
    )

@router.put("/{skill_id}", response_model=APIResponse)
def update_skill(
    *,
    skill_id: UUID,
    skill_in: SkillUpdate,
    skill_service: SkillService = Depends(get_skill_service),
    current_user: User = Depends(deps.required_admin_only),
) -> APIResponse:
    """
    Update an existing skill. Admin access only.
    """
    skill = skill_service.get(db=skill_service.db, id=skill_id)
    if not skill:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Skill not found",
        )
    # Check if updating name and if new name already exists
    if skill_in.name and skill_in.name != skill.name:
        existing_skill = skill_service.db.query(skill_service.model).filter(
            skill_service.model.name.ilike(skill_in.name),
            skill_service.model.id != skill_id,
        ).first()
        if existing_skill:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="A skill with this name already exists",
            )
    updated_skill = skill_service.update(
        db=skill_service.db, db_obj=skill, obj_in=skill_in
    )
    return APIResponse(
        status=200,
        message="Skill updated successfully.",
        data={
            "id": str(updated_skill.id),
            "name": updated_skill.name,
            "category": updated_skill.category,
        },
    )

@router.delete("/{skill_id}", response_model=APIResponse)
def delete_skill(
    *,
    skill_id: UUID,
    skill_service: SkillService = Depends(get_skill_service),
    current_user: User = Depends(deps.required_admin_only),
) -> APIResponse:
    """
    Delete a skill. Admin access only.
    """
    skill = skill_service.get(db=skill_service.db, id=skill_id)
    if not skill:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Skill not found",
        )
    # Check if skill is being used by any jobseekers or job postings
    if skill.jobseekers or skill.job_postings:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Cannot delete skill that is currently being used by jobseekers or job postings",
        )
    skill_service.remove(db=skill_service.db, id=skill_id)
    return APIResponse(
        status=200,
        message="Skill deleted successfully.",
        data={},
    )
```

---

## Service (business logic)


```python
from typing import List
from fastapi import Depends
from sqlalchemy.orm import Session

from app.db.database import get_db
from app.db.models.skill import Skill
from app.schemas.skill import SkillCreate, SkillUpdate, Skill as SkillSchema
from app.services.base import CRUDBase

class SkillService(CRUDBase[Skill, SkillCreate, SkillUpdate]):
    def __init__(self, db: Session):
        super().__init__(model=Skill)
        self.db = db

    def get_all_skills(self) -> List[Skill]:
        """Get all skills ordered by category and name"""
        return self.db.query(Skill).order_by(Skill.category, Skill.name).all()

def get_skill_service(db: Session = Depends(get_db)) -> SkillService:
    return SkillService(db=db)
```

```python
# Excerpt from app/services/base.py – generic CRUD used by SkillService

class CRUDBase(Generic[ModelType, CreateSchemaType, UpdateSchemaType]):
    def __init__(self, model: Type[ModelType]):
        self.model = model

    def get(self, db: Session, id: Any) -> Optional[ModelType]:
        return db.query(self.model).filter(self.model.id == id).first()

    def get_multi(self, db: Session, *, skip: int = 0, limit: int = 100) -> List[ModelType]:
        return db.query(self.model).offset(skip).limit(limit).all()

    def create(self, db: Session, *, obj_in: CreateSchemaType) -> ModelType:
        obj_in_data = jsonable_encoder(obj_in)
        db_obj = self.model(**obj_in_data)
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def update(
        self,
        db: Session,
        *,
        db_obj: ModelType,
        obj_in: Union[UpdateSchemaType, Dict[str, Any]],
    ) -> ModelType:
        obj_data = jsonable_encoder(db_obj)
        if isinstance(obj_in, dict):
            update_data = obj_in
        else:
            update_data = obj_in.model_dump(exclude_unset=True)
        for field in obj_data:
            if field in update_data:
                setattr(db_obj, field, update_data[field])
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def remove(self, db: Session, *, id: int) -> Optional[ModelType]:
        obj = db.query(self.model).get(id)
        if obj:
            db.delete(obj)
            db.commit()
        return obj
```

---

## Response wrapper

All endpoints return this Pydantic model so the API has a consistent envelope (`status`, `message`, `data`, `error`).


```python
from typing import Any, Dict, Union
from pydantic import BaseModel

class APIResponse(BaseModel):
    status: int
    message: str = ""
    data: Dict[str, Any] | list[Dict[str, Any]] | Any = {}
    error: Union[str, Dict[str, Any]] = ""
```
