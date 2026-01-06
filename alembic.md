### Step-by-Step Alembic Setup for FastAPI

#### **Step 1: Install Alembic**

Alembic must be installed to handle database migrations. If it's not installed, you can run one of the following commands:

- Direct installation:
  ```bash
  pip install alembic
  ```

- Alternatively, add Alembic to your `requirements.txt` and install:
  ```bash
  pip install -r requirements.txt
  ```

#### **Step 2: Initialize Alembic Directory**

Run the command:
```bash
alembic init app/models/migrations
```

This will initialize the Alembic directory structure under `app/models/migrations/`. It’s a specific directory derived from your repo to ensure migrations stay close to your models.

#### **Step 3: Update Alembic Configuration**

The Alembic configuration file is located at the project root: `alembic.ini`. Open and ensure that the `script_location` corresponds to the migration folder:
```
script_location = app/models/migrations
```

This keeps Alembic aware of the folder where migration files are stored.

---

#### **Step 4: Edit Alembic `env.py` for SQLAlchemy Metadata**

The `env.py` file in `app/models/migrations/` needs to be edited to let Alembic know about your SQLAlchemy models. You’ll particularly add your `Base.metadata` from SQLAlchemy. For example:
```python
from app.core.database import Base

# Make following assignment
target_metadata = Base.metadata
```

This `target_metadata` ensures that Alembic can generate migration scripts automatically based on the defined models.

---

#### **Step 5: Automatically Generate a Migration**

After updating the models or schemas in your FastAPI application, you can create new migrations to reflect these changes.

Run the following command:
```bash
alembic -c alembic.ini revision --autogenerate -m "Migration Message Here"
```
This command does the following:
- Looks for changes between your database and models.
- Generates a new migration script under `app/models/migrations/versions/`.
- Specifies a message ("Migration Message Here") to describe the nature of the migration.

---

#### **Step 6: Apply the Migration**

To apply the generated migration against the database, run:
```bash
alembic -c alembic.ini upgrade head
```

This command applies all pending migrations, bringing the database state to the latest version as defined by your models.

---

### Why Each Step is Important

- **Installation:** Ensures you have Alembic on your system as a dependency.
- **Initialization:** This step lays out the structure needed for migrations, creating the `migrations/` folder in your repository.
- **Configuration:** Uses `alembic.ini`, ensuring the migration scripts can be located and applied correctly.
- **Editing `env.py`:** Sets up the connection between the database and your application’s models using SQLAlchemy metadata.
- **Generating Migrations:** Keeps database schema in sync with the application, avoiding mismatches or manual migrations.
- **Applying Migrations:** Writes changes to the actual database, effectively updating schema.

---

**Final Notes:**
This structure in your repository is clear and ensures database migrations are well-organized, with essential configuration placed near their domain-specific models. For further details, refer to the full [ALEMBIC_SETUP.md document](https://github.com/Nik35/FastAPIStructure/blob/6f0d2d8eb77f28848df26b5aa2de71da1f73d1b4/ALEMBIC_SETUP.md#L1-L75).