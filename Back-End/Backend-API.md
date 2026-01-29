**Backend API: CRUD Example (Skills)**

**Base path:** /api/v1/skills

**API**

GET /api/v1/skills/ | List all skills (ordered by category, name).

POST /api/v1/skills/ | Create a skill.

PUT /api/v1/skills/{skill\_id} | Update a skill.

DELETE /api/v1/skills/{skill\_id} || Delete a skill

**Response**:All endpoints return the project’s standard APIResponse:

json

{

"status": 200,

"message": "Skills retrieved successfully.",

"data": \[...\],

"error": ""

}

**Model (DB layer)**

from sqlalchemy import Column, Integer, String, Table, ForeignKey

from sqlalchemy.orm import relationship

from sqlalchemy.dialects.postgresql import UUID

import uuid

from app.db.database import Base

\# Association table for JobseekerProfile and Skill

jobseeker\_skills = Table(

"jobseeker\_skills",

Base.metadata,

Column("jobseeker\_profile\_id", UUID(as\_uuid=True), ForeignKey("jobseeker\_profiles.id", ondelete="CASCADE"), primary\_key=True),

Column("skill\_id", UUID(as\_uuid=True), ForeignKey("skills.id", ondelete="CASCADE"), primary\_key=True),

)

\# Association table for JobPosting and Skill

job\_posting\_skills = Table(

"job\_posting\_skills",

Base.metadata,

Column("job\_posting\_id", UUID(as\_uuid=True), ForeignKey("job\_postings.id", ondelete="CASCADE"), primary\_key=True),

Column("skill\_id", UUID(as\_uuid=True), ForeignKey("skills.id", ondelete="CASCADE"), primary\_key=True),

)

class Skill(Base):

\_\_tablename\_\_ = "skills"

id = Column(UUID(as\_uuid=True), primary\_key=True, default=uuid.uuid4, index=True)

name = Column(String, unique=True, index=True, nullable=False)

category = Column(String, index=True, nullable=True) # e.g., 'Programming Language', 'Framework', 'Soft Skill'

\# Relationships to JobseekerProfile and JobPosting are defined in those models

jobseekers = relationship("JobseekerProfile", secondary=jobseeker\_skills, back\_populates="skills")

job\_postings = relationship("JobPosting", secondary=job\_posting\_skills, back\_populates="skills")

**Schema (request/response)**

\# Base fields shared by Create, Update, and response

class SkillBase(BaseModel):

name: str

category: Optional\[str\] = None # e.g. "Programming Language", "Framework", "Soft Skill"

\# Request body for PUT /api/v1/skills/{skill\_id} — all fields optional for partial updates

class SkillUpdate(SkillBase):

name: Optional\[str\] = None

category: Optional\[str\] = None

\# Response/serialization schema; includes id from DB. Config allows building from ORM (Skill model)

class Skill(SkillBase):

id: UUID

class Config:

from\_attributes = True # Pydantic v2: allow construction from SQLAlchemy model instances

**API (Router)**

from typing import List

from fastapi import APIRouter, Depends, HTTPException, status

from uuid import UUID

from app.api import deps

from app.db.models.user import User

from app.schemas.skill import Skill, SkillCreate, SkillUpdate

from app.services.skill import SkillService, get\_skill\_service

from app.utils.response\_format import APIResponse

router = APIRouter()

@router.get("/", response\_model=APIResponse)

def get\_all\_skills(

skill\_service: SkillService = Depends(get\_skill\_service),

) -> APIResponse:

"""

Get all available skills with their IDs and categories.

Open to all users.

"""

skills = skill\_service.get\_all\_skills()

return APIResponse(

status=200,

message="Skills retrieved successfully.",

data=\[

{"id": str(skill.id), "name": skill.name, "category": skill.category}

for skill in skills

\],

)

@router.post("/", response\_model=APIResponse, status\_code=status.HTTP\_201\_CREATED)

def create\_skill(

\*,

skill\_in: SkillCreate,

skill\_service: SkillService = Depends(get\_skill\_service),

current\_user: User = Depends(deps.required\_admin\_only),

) -> APIResponse:

"""

Create a new skill. Admin access only.

"""

\# Check if skill with same name already exists

existing\_skill = skill\_service.db.query(skill\_service.model).filter(

skill\_service.model.name.ilike(skill\_in.name)

).first()

if existing\_skill:

raise HTTPException(

status\_code=status.HTTP\_400\_BAD\_REQUEST,

detail="A skill with this name already exists",

)

skill = skill\_service.create(db=skill\_service.db, obj\_in=skill\_in)

return APIResponse(

status=status.HTTP\_201\_CREATED,

message="Skill created successfully.",

data={"id": str(skill.id), "name": skill.name, "category": skill.category},

)

@router.put("/{skill\_id}", response\_model=APIResponse)

def update\_skill(

\*,

skill\_id: UUID,

skill\_in: SkillUpdate,

skill\_service: SkillService = Depends(get\_skill\_service),

current\_user: User = Depends(deps.required\_admin\_only),

) -> APIResponse:

"""

Update an existing skill. Admin access only.

"""

skill = skill\_service.get(db=skill\_service.db, id=skill\_id)

if not skill:

raise HTTPException(

status\_code=status.HTTP\_404\_NOT\_FOUND,

detail="Skill not found",

)

\# Check if updating name and if new name already exists

if skill\_in.name and skill\_in.name != skill.name:

existing\_skill = skill\_service.db.query(skill\_service.model).filter(

skill\_service.model.name.ilike(skill\_in.name),

skill\_service.model.id != skill\_id,

).first()

if existing\_skill:

raise HTTPException(

status\_code=status.HTTP\_400\_BAD\_REQUEST,

detail="A skill with this name already exists",

)

updated\_skill = skill\_service.update(

db=skill\_service.db, db\_obj=skill, obj\_in=skill\_in

)

return APIResponse(

status=200,

message="Skill updated successfully.",

data={

"id": str(updated\_skill.id),

"name": updated\_skill.name,

"category": updated\_skill.category,

},

)

@router.delete("/{skill\_id}", response\_model=APIResponse)

def delete\_skill(

\*,

skill\_id: UUID,

skill\_service: SkillService = Depends(get\_skill\_service),

current\_user: User = Depends(deps.required\_admin\_only),

) -> APIResponse:

"""

Delete a skill. Admin access only.

"""

skill = skill\_service.get(db=skill\_service.db, id=skill\_id)

if not skill:

raise HTTPException(

status\_code=status.HTTP\_404\_NOT\_FOUND,

detail="Skill not found",

)

\# Check if skill is being used by any jobseekers or job postings

if skill.jobseekers or skill.job\_postings:

raise HTTPException(

status\_code=status.HTTP\_400\_BAD\_REQUEST,

detail="Cannot delete skill that is currently being used by jobseekers or job postings",

)

skill\_service.remove(db=skill\_service.db, id=skill\_id)

return APIResponse(

status=200,

message="Skill deleted successfully.",

data={},

)

\`\`\`

\---

**6\. Service (business logic)**

from typing import List

from fastapi import Depends

from sqlalchemy.orm import Session

from app.db.database import get\_db

from app.db.models.skill import Skill

from app.schemas.skill import SkillCreate, SkillUpdate, Skill as SkillSchema

from app.services.base import CRUDBase

class SkillService(CRUDBase\[Skill, SkillCreate, SkillUpdate\]):

def \_\_init\_\_(self, db: Session):

super().\_\_init\_\_(model=Skill)

self.db = db

def get\_all\_skills(self) -> List\[Skill\]:

"""Get all skills ordered by category and name"""

return self.db.query(Skill).order\_by(Skill.category, Skill.name).all()

def get\_skill\_service(db: Session = Depends(get\_db)) -> SkillService:

return SkillService(db=db)

\`\`\`

\*\*CRUDBase\*\* (from \`app/services/base.py\`) provides the generic CRUD used by the router:

\`\`\`python

\# Excerpt from app/services/base.py – generic CRUD used by SkillService

class CRUDBase(Generic\[ModelType, CreateSchemaType, UpdateSchemaType\]):

def \_\_init\_\_(self, model: Type\[ModelType\]):

self.model = model

def get(self, db: Session, id: Any) -> Optional\[ModelType\]:

return db.query(self.model).filter(self.model.id == id).first()

def get\_multi(self, db: Session, \*, skip: int = 0, limit: int = 100) -> List\[ModelType\]:

return db.query(self.model).offset(skip).limit(limit).all()

def create(self, db: Session, \*, obj\_in: CreateSchemaType) -> ModelType:

obj\_in\_data = jsonable\_encoder(obj\_in)

db\_obj = self.model(\*\*obj\_in\_data)

db.add(db\_obj)

db.commit()

db.refresh(db\_obj)

return db\_obj

def update(

self,

db: Session,

\*,

db\_obj: ModelType,

obj\_in: Union\[UpdateSchemaType, Dict\[str, Any\]\],

) -> ModelType:

obj\_data = jsonable\_encoder(db\_obj)

if isinstance(obj\_in, dict):

update\_data = obj\_in

else:

update\_data = obj\_in.model\_dump(exclude\_unset=True)

for field in obj\_data:

if field in update\_data:

setattr(db\_obj, field, update\_data\[field\])

db.add(db\_obj)

db.commit()

db.refresh(db\_obj)

return db\_obj

def remove(self, db: Session, \*, id: int) -> Optional\[ModelType\]:

obj = db.query(self.model).get(id)

if obj:

db.delete(obj)

db.commit()

return obj

**Response wrapper**

All endpoints return this Pydantic model so the API has a consistent envelope (\`status\`, \`message\`, \`data\`, \`error\`).

from typing import Any, Dict, Union

from pydantic import BaseModel

class APIResponse(BaseModel):

status: int

message: str = ""

data: Dict\[str, Any\] | list\[Dict\[str, Any\]\] | Any = {}

error: Union\[str, Dict\[str, Any\]\] = ""