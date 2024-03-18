# ЛР 1. Реализация серверного приложения FastAPI.

# Модели

<br>

## Поездка - Trip
*trip_models.py*
```
class StatusType(Enum):
    open = "open"
    closed = "closed"
    cancelled = "cancelled"


class TripDefault(SQLModel):
    status: StatusType = StatusType.open
    member_capacity: Optional[int] = Field(default=None, ge=0)
    

class Trip(TripDefault, table=True):
    id: int = Field(default=None, primary_key=True)
    members: Optional[List["UserTripLink"]] = Relationship(back_populates="trip", 
                                                           sa_relationship_kwargs={"cascade": "all, delete"})
    steps: Optional[List["Step"]] = Relationship(back_populates="trip", 
                                                 sa_relationship_kwargs={"cascade": "all, delete"})

class TripDetailed(TripDefault):
    members: Optional[List["UserTripLinkUsers"]] = None
    steps: Optional[List["StepDetailed"]] = None
```

<br>

## Пользователь - User
*user_models.py*
```
class GenderType(Enum):
    undefined = "-"
    female = "f"
    male = "m"


class UserDefault(SQLModel):
    username: str = Field(index=True, unique=True)
    first_name: str
    last_name: str
    gender: GenderType = "-"
    age: int = Field(ge=0, le=130)
    telephone: str = Field(regex="\+?\d{7,11}")
    email: EmailStr = Field(sa_type=AutoString)
    bio: Optional[str] = ""


class User(UserDefault, table=True):
    id: int = Field(default=None, primary_key=True)
    password: str = Field(min_length=8, max_length=60)
    registered: datetime.datetime = datetime.datetime.now()
    is_admin: bool = False
    trips: Optional[List["UserTripLink"]] = Relationship(back_populates="user", 
                                                         sa_relationship_kwargs={"cascade": "all, delete"})


class UserInput(SQLModel):
    username: str = Field(index=True, unique=True)
    password: str = Field(min_length=8, max_length=60)
    password2: str = Field(min_length=8, max_length=60)
    first_name: str
    last_name: str
    gender: GenderType = "-"
    age: int = Field(ge=0, le=130)
    telephone: str = Field(regex="\+?\d{7,11}")
    email: EmailStr = Field(sa_type=AutoString)
    bio: Optional[str] = ""

    @validator('password2')
    def check_match(cls, pwd, values, **kwargs):
        if 'password' in values and pwd != values['password']:
            raise ValueError("passwords don't match")
        return pwd


class UserLogin(SQLModel):
    username: str
    password: str


class UserPwd(SQLModel):
    old_password: str = Field(min_length=8, max_length=60)
    new_password: str = Field(min_length=8, max_length=60)
    new_password2: str = Field(min_length=8, max_length=60)
```

<br>

## Связь Пользователь-Поездка - UserTripLink
Ассоциативная сущность, соединяющая пользователя с поездкой
<br><br>
*usertriplink_models.py*
```
class UserTripLinkDefault(SQLModel):
    __table_args__ = (
        UniqueConstraint("trip_id", "user_id", name="unique pair of ids"),
    )
    user_id: Optional[int] = Field(sa_column=Column(Integer,
        ForeignKey("user.id", ondelete='CASCADE'), default=None))
    trip_id: Optional[int] = Field(sa_column=Column(Integer,
        ForeignKey("trip.id", ondelete='CASCADE'), default=None))
    role: Optional[str]


class UserTripLink(UserTripLinkDefault, table=True):
    id: int = Field(default=None, primary_key=True)
    user: "User" = Relationship(back_populates="trips")
    trip: "Trip" = Relationship(back_populates="members")


class UserTripLinkTrips(SQLModel):
    role: Optional[str]
    trip: "TripDetailed" = None


class UserTripLinkUsers(SQLModel):
    role: Optional[str]
    user: "UserDefault" = None
```

<br>

## Стоянка - Stay
*stay_models.py*
```
class AccomodationType(Enum):
    hotel = "hotel"
    hostel = "hostel"
    apartments = "apartments"
    couchsurfing = "couchsurfing"
    tent = "tent"


class StayDefault(SQLModel):
    location: str
    address: str
    accomodation: AccomodationType


class Stay(StayDefault, table=True):
    id: int = Field(default=None, primary_key=True)
    steps: Optional[List["Step"]] = Relationship(back_populates="stay",
                                                 sa_relationship_kwargs={"cascade": "all, delete"})
```

<br>

## Переезд - Transition
*transiition_models.py*
```
class TransportType(Enum):
    plane = "plane"
    train = "train"
    ship = "ship"
    ferry = "ferry"
    bus = "bus"
    car = "car"
    motorbike = "motorbike"
    bycicle = "bycicle"
    walking = "walking"
    hitchhiking = "hitchhiking"


class TransitionDefault(SQLModel):
    location_from: str
    location_to: str
    transport: TransportType


class Transition(TransitionDefault, table=True):
    id: int = Field(default=None, primary_key=True)
    steps: Optional[List["Step"]] = Relationship(back_populates="transition", 
                                                 sa_relationship_kwargs={"cascade": "all, delete"})
```

<br>

## Этап - Step
Ассоциативная сущность, соединяющая поездку со стоянкой или переездом
<br><br>
*step_models.py*
```
class StepDefault(SQLModel):
    trip_id: Optional[int] = Field(sa_column=Column(Integer,
        ForeignKey("trip.id", ondelete='CASCADE'), default=None))
    date_from: datetime = Field(ge=datetime.now())
    date_to: datetime = Field(ge=datetime.now())
    est_price: float = Field(default=0, ge=0)
    stay_id: Optional[int] = Field(sa_column=Column(Integer,
        ForeignKey("stay.id", ondelete='CASCADE'), default=None))
    #OR
    transition_id: Optional[int] = Field(sa_column=Column(Integer,
        ForeignKey("transition.id", ondelete='CASCADE'), default=None))

    @validator('date_to')
    def from_lt_to(cls, date, values, **kwargs):
            if 'date_from' in values and date < values['date_from']:
                raise ValueError("date_to should be later than date_from")
            return date
    

class Step(StepDefault, table=True):
    id: int = Field(default=None, primary_key=True)
    trip: "Trip" = Relationship(back_populates="steps")
    stay: Optional["Stay"] = Relationship(back_populates="steps")
    transition: Optional["Transition"] = Relationship(back_populates="steps")


class StepDetailed(StepDefault):
    trip: "Trip" = None
    stay: Optional["Stay"] = None
    transition: Optional["Transition"] = None
```

<br>