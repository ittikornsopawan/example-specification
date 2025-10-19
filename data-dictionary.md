# Data Dictionary: Microservice-IAM

## m_parameters

Master table to store system parameters. Supports multiple languages, active/inactive status, logical deletion, and effective/expiration periods.

### Table Columns

| Key | Column Name  | Data Type    | Default           | Description                                                            |
| --- | ------------ | ------------ | ----------------- | ---------------------------------------------------------------------- |
| PK  | id           | UUID         | GEN_RANDOM_UUID() | Primary key of the record                                              |
| FK  | created_by   | UUID         |                   | References the creator from `authentication.t_users(id)`               |
|     | created_at   | TIMESTAMP    | CURRENT_TIMESTAMP | Record creation timestamp                                              |
| FK  | updated_by   | UUID         |                   | References the last updater from `authentication.t_users(id)`          |
|     | updated_at   | TIMESTAMP    |                   | Timestamp of the last update                                           |
|     | is_active    | BOOLEAN      | FALSE             | Active status of the parameter                                         |
| FK  | inactive_by  | UUID         |                   | References the user who deactivated the parameter                      |
|     | inactive_at  | TIMESTAMP    |                   | Timestamp when the parameter was deactivated                           |
|     | is_deleted   | BOOLEAN      | FALSE             | Logical deletion flag                                                  |
| FK  | deleted_by   | UUID         |                   | References the user who deleted the parameter                          |
|     | deleted_at   | TIMESTAMP    |                   | Timestamp when the parameter was deleted                               |
|     | effective_at | TIMESTAMP    | CURRENT_TIMESTAMP | The effective start date of the parameter                              |
|     | expires_at   | TIMESTAMP    |                   | The expiration date of the parameter (must be later than effective_at) |
|     | key          | VARCHAR(128) |                   | Parameter name (unique identifier)                                     |
|     | language     | VARCHAR(16)  |                   | Language code for the parameter value (e.g., 'en', 'th')               |
|     | value        | TEXT         |                   | Parameter value                                                        |

```sql
CREATE TABLE IF NOT EXISTS m_parameters
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    effective_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP CHECK (expires_at IS NOT NULL AND expires_at > effective_at),

    key VARCHAR(128) NOT NULL,
    language VARCHAR(16),
    value TEXT
);
CREATE INDEX IF NOT EXISTS idx_m_parameters_key ON m_parameters(key);
```

### Table Constraints

| Constraint Type | Constraint Name      | Description                                                       |
| --------------- | -------------------- | ----------------------------------------------------------------- |
| PRIMARY KEY     | id                   | Defines `id` as the primary key                                   |
| FOREIGN KEY     | created_by           | References `authentication.t_users(id)` for the creator           |
| FOREIGN KEY     | updated_by           | References `authentication.t_users(id)` for the last updater      |
| FOREIGN KEY     | inactive_by          | References `authentication.t_users(id)` for deactivation          |
| FOREIGN KEY     | deleted_by           | References `authentication.t_users(id)` for deletion              |
| CHECK           | expires_at           | Ensures `expires_at` is later than `effective_at` if not NULL     |
| INDEX           | idx_m_parameters_key | Creates an index on the `key` column to improve query performance |

## t_logs

Table to store audit logs for tracking actions in the system, including who created the log and which table the log refers to.

### Table Columns

| Key | Column Name | Data Type    | Default           | Description                                              |
| --- | ----------- | ------------ | ----------------- | -------------------------------------------------------- |
| PK  | id          | UUID         | GEN_RANDOM_UUID() | Primary key of the log record                            |
| FK  | created_by  | UUID         |                   | References the creator from `authentication.t_users(id)` |
|     | created_at  | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the log was created                       |
|     | code        | VARCHAR(32)  |                   | Unique code identifier for the log entry                 |
|     | table_name  | VARCHAR(128) |                   | Name of the table associated with the log                |

```sql
CREATE TABLE IF NOT EXISTS audit_trail.t_logs
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    code VARCHAR(32) NOT NULL UNIQUE,
    table_name VARCHAR(128) NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_t_logs_table_name ON audit_trail.t_logs(table_name);
```

### Table Constraints

| Constraint Type | Constraint Name       | Description                                                   |
| --------------- | --------------------- | ------------------------------------------------------------- |
| PRIMARY KEY     | id                    | Defines `id` as the primary key                               |
| FOREIGN KEY     | created_by            | References `authentication.t_users(id)` for the log creator   |
| UNIQUE          | code                  | Ensures `code` is unique for each log entry                   |
| INDEX           | idx_t_logs_table_name | Creates an index on `table_name` to improve query performance |

## t_log_properties

Table to store detailed changes (properties) for each audit log entry, including which column was changed and its old and new values.

### Table Columns

| Key | Column Name | Data Type    | Default           | Description                                                   |
| --- | ----------- | ------------ | ----------------- | ------------------------------------------------------------- |
| PK  | id          | UUID         | GEN_RANDOM_UUID() | Primary key of the log property record                        |
| FK  | created_by  | UUID         |                   | References the creator from `authentication.t_users(id)`      |
|     | created_at  | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the log property was created                   |
| FK  | log_id      | UUID         |                   | References the parent log entry from `audit_trail.t_logs(id)` |
|     | column_name | VARCHAR(128) |                   | Name of the column that was changed                           |
|     | current     | TEXT         |                   | Current (old) value of the column before change               |
|     | new         | TEXT         |                   | New value of the column after change                          |

```sql
CREATE TABLE IF NOT EXISTS audit_trail.t_log_properties
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    log_id UUID NOT NULL REFERENCES audit_trail.t_logs(id),

    column_name VARCHAR(128) NOT NULL,
    current TEXT NOT NULL,
    new TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_t_log_properties_log_id ON audit_trail.t_log_properties(log_id);
```

### Table Constraints

| Constraint Type | Constraint Name             | Description                                                          |
| --------------- | --------------------------- | -------------------------------------------------------------------- |
| PRIMARY KEY     | id                          | Defines `id` as the primary key                                      |
| FOREIGN KEY     | created_by                  | References `authentication.t_users(id)` for the log property creator |
| FOREIGN KEY     | log_id                      | References `audit_trail.t_logs(id)` to link each property to its log |
| INDEX           | idx_t_log_properties_log_id | Creates an index on `log_id` to improve query performance            |

## m_algorithms

Master table to store algorithm definitions, including metadata, status, versioning, and required keys for execution.

### Table Columns

| Key | Column Name  | Data Type    | Default           | Description                                                        |
| --- | ------------ | ------------ | ----------------- | ------------------------------------------------------------------ |
| PK  | id           | UUID         | GEN_RANDOM_UUID() | Primary key of the algorithm record                                |
| FK  | created_by   | UUID         |                   | References the creator from `authentication.t_users(id)`           |
|     | created_at   | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the record was created                              |
| FK  | updated_by   | UUID         |                   | References the last updater from `authentication.t_users(id)`      |
|     | updated_at   | TIMESTAMP    |                   | Timestamp of the last update                                       |
|     | is_active    | BOOLEAN      | FALSE             | Active status of the algorithm                                     |
| FK  | inactive_by  | UUID         |                   | References the user who deactivated the algorithm                  |
|     | inactive_at  | TIMESTAMP    |                   | Timestamp when the algorithm was deactivated                       |
|     | is_deleted   | BOOLEAN      | FALSE             | Logical deletion flag                                              |
| FK  | deleted_by   | UUID         |                   | References the user who deleted the algorithm                      |
|     | deleted_at   | TIMESTAMP    |                   | Timestamp when the algorithm was deleted                           |
|     | effective_at | TIMESTAMP    | CURRENT_TIMESTAMP | Effective start date of the algorithm                              |
|     | expires_at   | TIMESTAMP    |                   | Expiration date of the algorithm (must be later than effective_at) |
|     | name         | VARCHAR(128) |                   | Name of the algorithm (unique)                                     |
|     | algorithm    | BYTEA        |                   | Binary data representing the algorithm                             |
|     | key_required | JSONB        |                   | JSON structure specifying required keys for the algorithm          |

```sql
CREATE TABLE IF NOT EXISTS algorithm.m_algorithms
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    effective_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP CHECK (expires_at IS NOT NULL AND expires_at > effective_at),

    name VARCHAR(128) NOT NULL,
    algorithm BYTEA NOT NULL,
    key_required JSONB
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_m_algorithms_name ON algorithm.m_algorithms(name);
```

### Table Constraints

| Constraint Type | Constraint Name       | Description                                                   |
| --------------- | --------------------- | ------------------------------------------------------------- |
| PRIMARY KEY     | id                    | Defines `id` as the primary key                               |
| FOREIGN KEY     | created_by            | References `authentication.t_users(id)` for the creator       |
| FOREIGN KEY     | updated_by            | References `authentication.t_users(id)` for the last updater  |
| FOREIGN KEY     | inactive_by           | References `authentication.t_users(id)` for deactivation      |
| FOREIGN KEY     | deleted_by            | References `authentication.t_users(id)` for deletion          |
| CHECK           | expires_at            | Ensures `expires_at` is later than `effective_at` if not NULL |
| UNIQUE          | idx_m_algorithms_name | Ensures `name` is unique                                      |

## t_key_types

Master table to store types of keys, including metadata, status, versioning, and descriptive information.

### Table Columns

| Key | Column Name  | Data Type    | Default           | Description                                                       |
| --- | ------------ | ------------ | ----------------- | ----------------------------------------------------------------- |
| PK  | id           | UUID         | GEN_RANDOM_UUID() | Primary key of the key type record                                |
| FK  | created_by   | UUID         |                   | References the creator from `authentication.t_users(id)`          |
|     | created_at   | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the record was created                             |
| FK  | updated_by   | UUID         |                   | References the last updater from `authentication.t_users(id)`     |
|     | updated_at   | TIMESTAMP    |                   | Timestamp of the last update                                      |
|     | is_active    | BOOLEAN      | FALSE             | Active status of the key type                                     |
| FK  | inactive_by  | UUID         |                   | References the user who deactivated the key type                  |
|     | inactive_at  | TIMESTAMP    |                   | Timestamp when the key type was deactivated                       |
|     | is_deleted   | BOOLEAN      | FALSE             | Logical deletion flag                                             |
| FK  | deleted_by   | UUID         |                   | References the user who deleted the key type                      |
|     | deleted_at   | TIMESTAMP    |                   | Timestamp when the key type was deleted                           |
|     | effective_at | TIMESTAMP    | CURRENT_TIMESTAMP | Effective start date of the key type                              |
|     | expires_at   | TIMESTAMP    |                   | Expiration date of the key type (must be later than effective_at) |
|     | name         | VARCHAR(128) |                   | Name of the key type (unique)                                     |
|     | title        | VARCHAR(512) |                   | Display title of the key type                                     |
|     | description  | TEXT         |                   | Description or details about the key type                         |

```sql
CREATE SCHEMA IF NOT EXISTS key.t_key_types
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    effective_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP CHECK (expires_at IS NOT NULL AND expires_at > effective_at),

    name VARCHAR(128) NOT NULL,
    title VARCHAR(512),
    description TEXT
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_t_key_types_name ON key.t_key_types(name);
```

### Table Constraints

| Constraint Type | Constraint Name      | Description                                                   |
| --------------- | -------------------- | ------------------------------------------------------------- |
| PRIMARY KEY     | id                   | Defines `id` as the primary key                               |
| FOREIGN KEY     | created_by           | References `authentication.t_users(id)` for the creator       |
| FOREIGN KEY     | updated_by           | References `authentication.t_users(id)` for the last updater  |
| FOREIGN KEY     | inactive_by          | References `authentication.t_users(id)` for deactivation      |
| FOREIGN KEY     | deleted_by           | References `authentication.t_users(id)` for deletion          |
| CHECK           | expires_at           | Ensures `expires_at` is later than `effective_at` if not NULL |
| UNIQUE          | idx_t_key_types_name | Ensures `name` is unique                                      |

## t_keys

Table to store keys, including metadata, status, versioning, and relation to key types.

### Table Columns

| Key | Column Name  | Data Type | Default           | Description                                                   |
| --- | ------------ | --------- | ----------------- | ------------------------------------------------------------- |
| PK  | id           | UUID      | GEN_RANDOM_UUID() | Primary key of the key record                                 |
| FK  | created_by   | UUID      |                   | References the creator from `authentication.t_users(id)`      |
|     | created_at   | TIMESTAMP | CURRENT_TIMESTAMP | Timestamp when the record was created                         |
| FK  | updated_by   | UUID      |                   | References the last updater from `authentication.t_users(id)` |
|     | updated_at   | TIMESTAMP |                   | Timestamp of the last update                                  |
|     | is_active    | BOOLEAN   | FALSE             | Active status of the key                                      |
| FK  | inactive_by  | UUID      |                   | References the user who deactivated the key                   |
|     | inactive_at  | TIMESTAMP |                   | Timestamp when the key was deactivated                        |
|     | is_deleted   | BOOLEAN   | FALSE             | Logical deletion flag                                         |
| FK  | deleted_by   | UUID      |                   | References the user who deleted the key                       |
|     | deleted_at   | TIMESTAMP |                   | Timestamp when the key was deleted                            |
|     | effective_at | TIMESTAMP | CURRENT_TIMESTAMP | Effective start date of the key                               |
|     | expires_at   | TIMESTAMP |                   | Expiration date of the key (must be later than effective_at)  |
| FK  | type_id      | UUID      |                   | References the key type from `key.t_key_types(id)`            |
|     | key          | BYTEA     |                   | Binary data of the key                                        |

```sql
CREATE TABLE IF NOT EXISTS key.t_keys
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    effective_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP CHECK (expires_at IS NOT NULL AND expires_at > effective_at),

    type_id UUID NOT NULL REFERENCES key.t_key_types(id),
    key BYTEA NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_t_keys_type_id ON key.t_keys(type_id);
```

### Table Constraints

| Constraint Type | Constraint Name    | Description                                                   |
| --------------- | ------------------ | ------------------------------------------------------------- |
| PRIMARY KEY     | id                 | Defines `id` as the primary key                               |
| FOREIGN KEY     | created_by         | References `authentication.t_users(id)` for the creator       |
| FOREIGN KEY     | updated_by         | References `authentication.t_users(id)` for the last updater  |
| FOREIGN KEY     | inactive_by        | References `authentication.t_users(id)` for deactivation      |
| FOREIGN KEY     | deleted_by         | References `authentication.t_users(id)` for deletion          |
| FOREIGN KEY     | type_id            | References `key.t_key_types(id)` to link each key to its type |
| INDEX           | idx_t_keys_type_id | Creates an index on `type_id` to improve query performance    |

## t_users

Master table to store system users, including metadata, status, and unique identifiers for login and reference.

### Table Columns

| Key | Column Name | Data Type    | Default           | Description                                                   |
| --- | ----------- | ------------ | ----------------- | ------------------------------------------------------------- |
| PK  | id          | UUID         | GEN_RANDOM_UUID() | Primary key of the user record                                |
| FK  | created_by  | UUID         |                   | References the creator from `authentication.t_users(id)`      |
|     | created_at  | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the record was created                         |
| FK  | updated_by  | UUID         |                   | References the last updater from `authentication.t_users(id)` |
|     | updated_at  | TIMESTAMP    |                   | Timestamp of the last update                                  |
|     | is_active   | BOOLEAN      | FALSE             | Active status of the user                                     |
| FK  | inactive_by | UUID         |                   | References the user who deactivated this user                 |
|     | inactive_at | TIMESTAMP    |                   | Timestamp when the user was deactivated                       |
|     | is_deleted  | BOOLEAN      | FALSE             | Logical deletion flag                                         |
| FK  | deleted_by  | UUID         |                   | References the user who deleted this user                     |
|     | deleted_at  | TIMESTAMP    |                   | Timestamp when the user was deleted                           |
|     | code        | VARCHAR(32)  |                   | Unique code identifier for the user                           |
|     | username    | VARCHAR(128) |                   | Unique username for login                                     |

```sql
CREATE TABLE IF NOT EXISTS authentication.t_users
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    code VARCHAR(32) NOT NULL UNIQUE,
    username VARCHAR(128) NOT NULL UNIQUE
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_t_users_code ON authentication.t_users(type_id);
CREATE UNIQUE INDEX IF NOT EXISTS idx_t_users_username ON authentication.t_users(username);
```

### Table Constraints

| Constraint Type | Constraint Name      | Description                                                  |
| --------------- | -------------------- | ------------------------------------------------------------ |
| PRIMARY KEY     | id                   | Defines `id` as the primary key                              |
| FOREIGN KEY     | created_by           | References `authentication.t_users(id)` for the creator      |
| FOREIGN KEY     | updated_by           | References `authentication.t_users(id)` for the last updater |
| FOREIGN KEY     | inactive_by          | References `authentication.t_users(id)` for deactivation     |
| FOREIGN KEY     | deleted_by           | References `authentication.t_users(id)` for deletion         |
| UNIQUE          | idx_t_users_code     | Ensures `code` is unique                                     |
| UNIQUE          | idx_t_users_username | Ensures `username` is unique                                 |

## t_user_authentications

Table to store authentication credentials for users, including algorithms, keys, password hash, and effective periods.

### Table Columns

| Key | Column Name    | Data Type | Default           | Description                                                     |
| --- | -------------- | --------- | ----------------- | --------------------------------------------------------------- |
| PK  | id             | UUID      | GEN_RANDOM_UUID() | Primary key of the authentication record                        |
| FK  | created_by     | UUID      |                   | References the creator from `authentication.t_users(id)`        |
|     | created_at     | TIMESTAMP | CURRENT_TIMESTAMP | Timestamp when the record was created                           |
| FK  | updated_by     | UUID      |                   | References the last updater from `authentication.t_users(id)`   |
|     | updated_at     | TIMESTAMP |                   | Timestamp of the last update                                    |
|     | is_active      | BOOLEAN   | FALSE             | Active status of the authentication record                      |
| FK  | inactive_by    | UUID      |                   | References the user who deactivated this record                 |
|     | inactive_at    | TIMESTAMP |                   | Timestamp when the record was deactivated                       |
|     | is_deleted     | BOOLEAN   | FALSE             | Logical deletion flag                                           |
| FK  | deleted_by     | UUID      |                   | References the user who deleted this record                     |
|     | deleted_at     | TIMESTAMP |                   | Timestamp when the record was deleted                           |
|     | effective_at   | TIMESTAMP | CURRENT_TIMESTAMP | Effective start date of this authentication record              |
|     | expires_at     | TIMESTAMP |                   | Expiration date (must be later than effective_at)               |
| FK  | user_id        | UUID      |                   | References the user from `authentication.t_users(id)`           |
|     | is_temporary   | BOOLEAN   | FALSE             | Indicates if this authentication is temporary                   |
| FK  | algorithm_id   | UUID      |                   | References the algorithm used from `algorithm.m_algorithms(id)` |
|     | algorithm_keys | JSONB     |                   | JSON structure storing keys required for the algorithm          |
|     | password_hash  | BYTEA     |                   | Hashed password for the user                                    |

```sql
CREATE TABLE IF NOT EXISTS authentication.t_user_authentications
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    effective_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP CHECK (expires_at IS NOT NULL AND expires_at > effective_at),

    user_id UUID NOT NULL REFERENCES authentication.t_users(id),
    is_temporary BOOLEAN NOT NULL DEFAULT FALSE,

    algorithm_id UUID NOT NULL REFERENCES algorithm.m_algorithms(id),
    algorithm_keys JSONB NOT NULL,

    password_hash BYTEA NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_t_user_authentications_user_id ON authentication.t_user_authentications(user_id);
```

### Table Constraints

| Constraint Type | Constraint Name                    | Description                                                    |
| --------------- | ---------------------------------- | -------------------------------------------------------------- |
| PRIMARY KEY     | id                                 | Defines `id` as the primary key                                |
| FOREIGN KEY     | created_by                         | References `authentication.t_users(id)` for the creator        |
| FOREIGN KEY     | updated_by                         | References `authentication.t_users(id)` for the last updater   |
| FOREIGN KEY     | inactive_by                        | References `authentication.t_users(id)` for deactivation       |
| FOREIGN KEY     | deleted_by                         | References `authentication.t_users(id)` for deletion           |
| FOREIGN KEY     | user_id                            | References `authentication.t_users(id)` for the user           |
| FOREIGN KEY     | algorithm_id                       | References `algorithm.m_algorithms(id)` for the algorithm used |
| CHECK           | expires_at                         | Ensures `expires_at` is later than `effective_at` if not NULL  |
| INDEX           | idx_t_user_authentications_user_id | Creates an index on `user_id` to improve query performance     |

## t_user_referrer_mappings

Table to map users to their referrers, tracking who referred whom in the system.

### Table Columns

| Key | Column Name | Data Type | Default           | Description                                                    |
| --- | ----------- | --------- | ----------------- | -------------------------------------------------------------- |
| PK  | id          | UUID      | GEN_RANDOM_UUID() | Primary key of the referrer mapping record                     |
| FK  | created_by  | UUID      |                   | References the creator from `authentication.t_users(id)`       |
|     | created_at  | TIMESTAMP | CURRENT_TIMESTAMP | Timestamp when the record was created                          |
| FK  | referrer_id | UUID      |                   | References the referrer user from `authentication.t_users(id)` |
| FK  | user_id     | UUID      |                   | References the referred user from `authentication.t_users(id)` |

```sql
CREATE TABLE IF NOT EXISTS authentication.t_user_referrer_mappings
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    referrer_id UUID NOT NULL REFERENCES authentication.t_users(id),
    user_id UUID NOT NULL REFERENCES authentication.t_users(id)
);
CREATE INDEX IF NOT EXISTS idx_t_user_referrer_mappings_user_id ON authentication.t_user_referrer_mappings(user_id);
```

### Table Constraints

| Constraint Type | Constraint Name | Description                                              |
| --------------- | --------------- | -------------------------------------------------------- |
| PRIMARY KEY     | id              | Defines `id` as the primary key                          |
| FOREIGN KEY     | created_by      | References `authentication.t_users(id)` for the creator  |
| FOREIGN KEY     | referrer_id     | References `authentication.t_users(id)` for the referrer |
| FOREIGN KEY     | user_id         | References `                                             |

## m_user_profiles

Table to store detailed user profile information, including names, display name, and date of birth, along with metadata and status.

### Table Columns

| Key | Column Name   | Data Type   | Default           | Description                                                   |
| --- | ------------- | ----------- | ----------------- | ------------------------------------------------------------- |
| PK  | id            | UUID        | GEN_RANDOM_UUID() | Primary key of the user profile record                        |
| FK  | created_by    | UUID        |                   | References the creator from `authentication.t_users(id)`      |
|     | created_at    | TIMESTAMP   | CURRENT_TIMESTAMP | Timestamp when the record was created                         |
| FK  | updated_by    | UUID        |                   | References the last updater from `authentication.t_users(id)` |
|     | updated_at    | TIMESTAMP   |                   | Timestamp of the last update                                  |
|     | is_active     | BOOLEAN     | FALSE             | Active status of the user profile                             |
| FK  | inactive_by   | UUID        |                   | References the user who deactivated this profile              |
|     | inactive_at   | TIMESTAMP   |                   | Timestamp when the profile was deactivated                    |
|     | is_deleted    | BOOLEAN     | FALSE             | Logical deletion flag                                         |
| FK  | deleted_by    | UUID        |                   | References the user who deleted this profile                  |
|     | deleted_at    | TIMESTAMP   |                   | Timestamp when the profile was deleted                        |
| FK  | user_id       | UUID        |                   | References the user from `authentication.t_users(id)`         |
|     | first_name    | BYTEA       |                   | Encrypted first name of the user                              |
|     | last_name     | BYTEA       |                   | Encrypted last name of the user                               |
|     | middle_name   | BYTEA       |                   | Encrypted middle name of the user (optional)                  |
|     | display_name  | VARCHAR(64) |                   | Display name of the user                                      |
|     | date_of_birth | DATE        |                   | Date of birth of the user                                     |

```sql
CREATE TABLE IF NOT EXISTS profile.m_user_profiles
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    user_id UUID NOT NULL REFERENCES authentication.t_users(id),

    first_name BYTEA NOT NULL,
    last_name BYTEA NOT NULL,
    middle_name BYTEA,

    display_name VARCHAR(64),
    date_of_birth DATE
);
CREATE INDEX IF NOT EXISTS idx_m_user_profiles_user_id ON attribute.m_user_profiles(user_id);
```

### Table Constraints

| Constraint Type | Constraint Name             | Description                                                  |
| --------------- | --------------------------- | ------------------------------------------------------------ |
| PRIMARY KEY     | id                          | Defines `id` as the primary key                              |
| FOREIGN KEY     | created_by                  | References `authentication.t_users(id)` for the creator      |
| FOREIGN KEY     | updated_by                  | References `authentication.t_users(id)` for the last updater |
| FOREIGN KEY     | inactive_by                 | References `authentication.t_users(id)` for deactivation     |
| FOREIGN KEY     | deleted_by                  | References `authentication.t_users(id)` for deletion         |
| FOREIGN KEY     | user_id                     | References `authentication.t_users(id)` for the user         |
| INDEX           | idx_m_user_profiles_user_id | Creates an index on `user_id` to improve query performance   |

## m_consent_types

Master table to store types of consents, including metadata, status, versioning, and descriptive information.

### Table Columns

| Key | Column Name | Data Type    | Default           | Description                                                   |
| --- | ----------- | ------------ | ----------------- | ------------------------------------------------------------- |
| PK  | id          | UUID         | GEN_RANDOM_UUID() | Primary key of the consent type record                        |
| FK  | created_by  | UUID         |                   | References the creator from `authentication.t_users(id)`      |
|     | created_at  | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the record was created                         |
| FK  | updated_by  | UUID         |                   | References the last updater from `authentication.t_users(id)` |
|     | updated_at  | TIMESTAMP    |                   | Timestamp of the last update                                  |
|     | is_active   | BOOLEAN      | FALSE             | Active status of the consent type                             |
| FK  | inactive_by | UUID         |                   | References the user who deactivated this consent type         |
|     | inactive_at | TIMESTAMP    |                   | Timestamp when the consent type was deactivated               |
|     | is_deleted  | BOOLEAN      | FALSE             | Logical deletion flag                                         |
| FK  | deleted_by  | UUID         |                   | References the user who deleted this consent type             |
|     | deleted_at  | TIMESTAMP    |                   | Timestamp when the consent type was deleted                   |
|     | name        | VARCHAR(128) |                   | Name of the consent type (unique)                             |
|     | title       | VARCHAR(512) |                   | Display title of the consent type                             |
|     | description | TEXT         |                   | Description or details about the consent type                 |

```sql
CREATE TABLE IF NOT EXISTS consent.m_consent_types
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    name VARCHAR(128) NOT NULL,
    title VARCHAR(512),
    description TEXT
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_m_consent_types_name ON consent.m_consent_types(name);
```

### Table Constraints

| Constraint Type | Constraint Name          | Description                                                  |
| --------------- | ------------------------ | ------------------------------------------------------------ |
| PRIMARY KEY     | id                       | Defines `id` as the primary key                              |
| FOREIGN KEY     | created_by               | References `authentication.t_users(id)` for the creator      |
| FOREIGN KEY     | updated_by               | References `authentication.t_users(id)` for the last updater |
| FOREIGN KEY     | inactive_by              | References `authentication.t_users(id)` for deactivation     |
| FOREIGN KEY     | deleted_by               | References `authentication.t_users(id)` for deletion         |
| UNIQUE          | idx_m_consent_types_name | Ensures `name` is unique                                     |

## m_consents

Table to store individual consent records, linked to consent types, with metadata, versioning, content, and status.

### Table Columns

| Key | Column Name | Data Type    | Default           | Description                                                    |
| --- | ----------- | ------------ | ----------------- | -------------------------------------------------------------- |
| PK  | id          | UUID         | GEN_RANDOM_UUID() | Primary key of the consent record                              |
| FK  | created_by  | UUID         |                   | References the creator from `authentication.t_users(id)`       |
|     | created_at  | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the record was created                          |
| FK  | updated_by  | UUID         |                   | References the last updater from `authentication.t_users(id)`  |
|     | updated_at  | TIMESTAMP    |                   | Timestamp of the last update                                   |
|     | is_active   | BOOLEAN      | FALSE             | Active status of the consent record                            |
| FK  | inactive_by | UUID         |                   | References the user who deactivated this record                |
|     | inactive_at | TIMESTAMP    |                   | Timestamp when the consent record was deactivated              |
|     | is_deleted  | BOOLEAN      | FALSE             | Logical deletion flag                                          |
| FK  | deleted_by  | UUID         |                   | References the user who deleted this record                    |
|     | deleted_at  | TIMESTAMP    |                   | Timestamp when the consent record was deleted                  |
| FK  | type_id     | UUID         |                   | References the consent type from `consent.m_consent_types(id)` |
|     | name        | VARCHAR(128) |                   | Name of the consent                                            |
|     | title       | VARCHAR(512) |                   | Display title of the consent                                   |
|     | content     | TEXT         |                   | Full content or text of the consent                            |
|     | version     | VARCHAR(8)   |                   | Version of the consent                                         |
|     | is_required | BOOLEAN      | FALSE             | Indicates whether this consent is mandatory                    |
|     | description | TEXT         |                   | Additional description or notes about the consent              |

```sql
CREATE TABLE IF NOT EXISTS consent.m_consents
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    type_id UUID NOT NULL REFERENCES consent.m_consent_types(id),

    name VARCHAR(128) NOT NULL,
    title VARCHAR(512),
    content TEXT NOT NULL,
    version VARCHAR(8) NOT NULL,
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    description TEXT
);
CREATE INDEX IF NOT EXISTS idx_m_consents_type_id ON consent.m_consents(type_id);
```

### Table Constraints

| Constraint Type | Constraint Name        | Description                                                               |
| --------------- | ---------------------- | ------------------------------------------------------------------------- |
| PRIMARY KEY     | id                     | Defines `id` as the primary key                                           |
| FOREIGN KEY     | created_by             | References `authentication.t_users(id)` for the creator                   |
| FOREIGN KEY     | updated_by             | References `authentication.t_users(id)` for the last updater              |
| FOREIGN KEY     | inactive_by            | References `authentication.t_users(id)` for deactivation                  |
| FOREIGN KEY     | deleted_by             | References `authentication.t_users(id)` for deletion                      |
| FOREIGN KEY     | type_id                | References `consent.m_consent_types(id)` to link each consent to its type |
| INDEX           | idx_m_consents_type_id | Creates an index on `type_id` to improve query performance                |

## t_user_consent_mappings

Table to store user consent records, tracking which user agreed or disagreed to specific consents and versions.

### Table Columns

| Key | Column Name | Data Type  | Default           | Description                                                    |
| --- | ----------- | ---------- | ----------------- | -------------------------------------------------------------- |
| PK  | id          | UUID       | GEN_RANDOM_UUID() | Primary key of the user consent record                         |
| FK  | created_by  | UUID       |                   | References the creator from `authentication.t_users(id)`       |
|     | created_at  | TIMESTAMP  | CURRENT_TIMESTAMP | Timestamp when the record was created                          |
| FK  | consent_id  | UUID       |                   | References the consent from `consent.m_consents(id)`           |
| FK  | type_id     | UUID       |                   | References the consent type from `consent.m_consent_types(id)` |
|     | version     | VARCHAR(8) |                   | Version of the consent agreed to                               |
| FK  | user_id     | UUID       |                   | References the user from `authentication.t_users(id)`          |
|     | result      | BOOLEAN    | FALSE             | Indicates whether the user agreed (TRUE) or not (FALSE)        |

```sql
CREATE TABLE IF NOT EXISTS consent.t_user_consent_mappings
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    consent_id UUID NOT NULL REFERENCES consent.m_consents(id),
    type_id UUID NOT NULL REFERENCES consent.m_consent_types(id),
    version VARCHAR(8) NOT NULL,
    user_id UUID NOT NULL REFERENCES authentication.t_users(id),
    result BOOLEAN NOT NULL DEFAULT FALSE
);
CREATE INDEX IF NOT EXISTS idx_t_user_consent_mappings_user_id ON consent.t_user_consent_mappings(user_id);
```

### Table Constraints

| Constraint Type | Constraint Name                     | Description                                                   |
| --------------- | ----------------------------------- | ------------------------------------------------------------- |
| PRIMARY KEY     | id                                  | Defines `id` as the primary key                               |
| FOREIGN KEY     | created_by                          | References `authentication.t_users(id)` for the creator       |
| FOREIGN KEY     | consent_id                          | References `consent.m_consents(id)` for the consent record    |
| FOREIGN KEY     | type_id                             | References `consent.m_consent_types(id)` for the consent type |
| FOREIGN KEY     | user_id                             | References `authentication.t_users(id)` for the user          |
| INDEX           | idx_t_user_consent_mappings_user_id | Creates an index on `user_id` to improve query performance    |

## m_attributes

Master table to store attributes used for authorization policies, including metadata, type, category, and display settings.

### Table Columns

| Key | Column Name  | Data Type    | Default           | Description                                                   |
| --- | ------------ | ------------ | ----------------- | ------------------------------------------------------------- |
| PK  | id           | UUID         | GEN_RANDOM_UUID() | Primary key of the attribute record                           |
| FK  | created_by   | UUID         |                   | References the creator from `authentication.t_users(id)`      |
|     | created_at   | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the record was created                         |
| FK  | updated_by   | UUID         |                   | References the last updater from `authentication.t_users(id)` |
|     | updated_at   | TIMESTAMP    |                   | Timestamp of the last update                                  |
|     | is_active    | BOOLEAN      | FALSE             | Active status of the attribute                                |
| FK  | inactive_by  | UUID         |                   | References the user who deactivated this attribute            |
|     | inactive_at  | TIMESTAMP    |                   | Timestamp when the attribute was deactivated                  |
|     | is_deleted   | BOOLEAN      | FALSE             | Logical deletion flag                                         |
| FK  | deleted_by   | UUID         |                   | References the user who deleted this attribute                |
|     | deleted_at   | TIMESTAMP    |                   | Timestamp when the attribute was deleted                      |
|     | is_parameter | BOOLEAN      | FALSE             | Indicates if the attribute is a system parameter              |
|     | is_required  | BOOLEAN      | FALSE             | Indicates if the attribute is required in policies            |
|     | is_display   | BOOLEAN      | FALSE             | Indicates if the attribute should be displayed in UI          |
|     | category     | VARCHAR(32)  | 'USER'            | Category of the attribute: USER, RESOURCE, or ENVIRONMENT     |
|     | key          | VARCHAR(128) |                   | Unique key identifier for the attribute                       |
|     | data_type    | VARCHAR(64)  |                   | Data type of the attribute (e.g., STRING, INTEGER, BOOLEAN)   |
|     | title        | VARCHAR(256) |                   | Display title of the attribute                                |
|     | description  | TEXT         |                   | Detailed description of the attribute                         |

```sql
CREATE TABLE IF NOT EXISTS authorization.m_attributes
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    is_parameter BOOLEAN NOT NULL DEFAULT FALSE,
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    is_display BOOLEAN NOT NULL DEFAULT FALSE,

    category VARCHAR(32) DEFAULT 'USER' CHECK (category IN ('USER','RESOURCE','ENVIRONMENT')),
    key VARCHAR(128) NOT NULL,
    data_type VARCHAR(64) NOT NULL,
    title VARCHAR(256),
    description TEXT
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_m_attributes_key ON authorization.m_attributes(key);
```

### Table Constraints

| Constraint Type | Constraint Name      | Description                                                       |
| --------------- | -------------------- | ----------------------------------------------------------------- |
| PRIMARY KEY     | id                   | Defines `id` as the primary key                                   |
| FOREIGN KEY     | created_by           | References `authentication.t_users(id)` for the creator           |
| FOREIGN KEY     | updated_by           | References `authentication.t_users(id)` for the last updater      |
| FOREIGN KEY     | inactive_by          | References `authentication.t_users(id)` for deactivation          |
| FOREIGN KEY     | deleted_by           | References `authentication.t_users(id)` for deletion              |
| UNIQUE          | idx_m_attributes_key | Ensures `key` is unique                                           |
| CHECK           | category             | Ensures `category` is one of 'USER', 'RESOURCE', or 'ENVIRONMENT' |

## t_policies

Table to store authorization policies, including metadata, effect, actions, resources, and conditions.

### Table Columns

| Key | Column Name     | Data Type    | Default           | Description                                                   |
| --- | --------------- | ------------ | ----------------- | ------------------------------------------------------------- |
| PK  | id              | UUID         | GEN_RANDOM_UUID() | Primary key of the policy record                              |
| FK  | created_by      | UUID         |                   | References the creator from `authentication.t_users(id)`      |
|     | created_at      | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the record was created                         |
| FK  | updated_by      | UUID         |                   | References the last updater from `authentication.t_users(id)` |
|     | updated_at      | TIMESTAMP    |                   | Timestamp of the last update                                  |
|     | is_active       | BOOLEAN      | FALSE             | Active status of the policy                                   |
| FK  | inactive_by     | UUID         |                   | References the user who deactivated this policy               |
|     | inactive_at     | TIMESTAMP    |                   | Timestamp when the policy was deactivated                     |
|     | is_deleted      | BOOLEAN      | FALSE             | Logical deletion flag                                         |
| FK  | deleted_by      | UUID         |                   | References the user who deleted this policy                   |
|     | deleted_at      | TIMESTAMP    |                   | Timestamp when the policy was deleted                         |
|     | name            | VARCHAR(128) |                   | Name of the policy (unique)                                   |
|     | description     | TEXT         |                   | Description or notes about the policy                         |
|     | code            | VARCHAR(32)  |                   | Unique code identifier for the policy                         |
|     | effect          | VARCHAR(16)  |                   | Effect of the policy, either 'ALLOW' or 'DENY'                |
|     | action          | VARCHAR(128) |                   | Action(s) the policy applies to                               |
|     | resource        | VARCHAR(256) |                   | Resource(s) the policy applies to                             |
|     | condition_logic | TEXT         |                   | Logic expression for conditions (optional)                    |

```sql
CREATE TABLE IF NOT EXISTS authorization.t_policies
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    name VARCHAR(128) NOT NULL,
    description TEXT,

    code VARCHAR(32) NOT NULL,
    effect VARCHAR(16) NOT NULL CHECK (effect IN ('ALLOW', 'DENY')),
    action VARCHAR(128) NOT NULL,
    resource VARCHAR(256) NOT NULL,
    condition_logic TEXT
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_t_policies_name ON authorization.t_policies(name);
```

### Table Constraints

| Constraint Type | Constraint Name     | Description                                                  |
| --------------- | ------------------- | ------------------------------------------------------------ |
| PRIMARY KEY     | id                  | Defines `id` as the primary key                              |
| FOREIGN KEY     | created_by          | References `authentication.t_users(id)` for the creator      |
| FOREIGN KEY     | updated_by          | References `authentication.t_users(id)` for the last updater |
| FOREIGN KEY     | inactive_by         | References `authentication.t_users(id)` for deactivation     |
| FOREIGN KEY     | deleted_by          | References `authentication.t_users(id)` for deletion         |
| UNIQUE          | idx_t_policies_name | Ensures `name` is unique                                     |
| CHECK           | effect              | Ensures `effect` is either 'ALLOW' or 'DENY'                 |

## t_policy_attribute_mappings

Table to map authorization policies to attributes, defining operators, expected values, and logic groups for policy evaluation.

### Table Columns

| Key | Column Name    | Data Type   | Default           | Description                                                         |
| --- | -------------- | ----------- | ----------------- | ------------------------------------------------------------------- |
| PK  | id             | UUID        | GEN_RANDOM_UUID() | Primary key of the policy-attribute mapping record                  |
| FK  | created_by     | UUID        |                   | References the creator from `authentication.t_users(id)`            |
|     | created_at     | TIMESTAMP   | CURRENT_TIMESTAMP | Timestamp when the record was created                               |
| FK  | policy_id      | UUID        |                   | References the policy from `authorization.t_policies(id)`           |
| FK  | attribute_id   | UUID        |                   | References the attribute from `authorization.m_attributes(id)`      |
|     | operator       | VARCHAR(32) |                   | Operator to evaluate the attribute (e.g., '=', '!=', '>', '<')      |
|     | expected_value | TEXT        |                   | Expected value of the attribute for the policy condition            |
|     | logic_group    | VARCHAR(16) | 'AND'             | Logic group for combining multiple conditions, either 'AND' or 'OR' |

```sql
CREATE TABLE IF NOT EXISTS authorization.t_policy_attribute_mappings
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    policy_id UUID NOT NULL REFERENCES authorization.t_policies(id),
    attribute_id UUID NOT NULL REFERENCES authorization.m_attributes(id),
    operator VARCHAR(32) NOT NULL,
    expected_value TEXT NOT NULL,
    logic_group VARCHAR(16) DEFAULT 'AND' CHECK (logic_group IN ('AND','OR'))
);
CREATE INDEX IF NOT EXISTS idx_t_policy_attribute_mappings_policy_id_attribute_id ON authorization.t_policy_attribute_mappings(policy_id, attribute_id);
```

### Table Constraints

| Constraint Type | Constraint Name                                        | Description                                                                              |
| --------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| PRIMARY KEY     | id                                                     | Defines `id` as the primary key                                                          |
| FOREIGN KEY     | created_by                                             | References `authentication.t_users(id)` for the creator                                  |
| FOREIGN KEY     | policy_id                                              | References `authorization.t_policies(id)` for the policy                                 |
| FOREIGN KEY     | attribute_id                                           | References `authorization.m_attributes(id)` for the attribute                            |
| CHECK           | logic_group                                            | Ensures `logic_group` is either 'AND' or 'OR'                                            |
| INDEX           | idx_t_policy_attribute_mappings_policy_id_attribute_id | Creates a composite index on `policy_id` and `attribute_id` to improve query performance |

## t_policy_decision_logs

Table to log the decisions of policy evaluations for users, including evaluated attributes, decision outcome, and reasons.

### Table Columns

| Key | Column Name          | Data Type    | Default           | Description                                                          |
| --- | -------------------- | ------------ | ----------------- | -------------------------------------------------------------------- |
| PK  | id                   | UUID         | GEN_RANDOM_UUID() | Primary key of the policy decision log record                        |
| FK  | created_by           | UUID         |                   | References the creator from `authentication.t_users(id)`             |
|     | created_at           | TIMESTAMP    | CURRENT_TIMESTAMP | Timestamp when the record was created                                |
| FK  | user_id              | UUID         |                   | References the user from `authentication.t_users(id)`                |
| FK  | policy_id            | UUID         |                   | References the policy from `authorization.t_policies(id)` (optional) |
|     | resource             | VARCHAR(256) |                   | Resource that the policy decision applies to                         |
|     | action               | VARCHAR(128) |                   | Action that the policy decision applies to                           |
|     | decision             | VARCHAR(16)  |                   | Outcome of the policy evaluation, either 'ALLOW' or 'DENY'           |
|     | evaluated_attributes | JSONB        |                   | JSON object storing evaluated attributes and their values            |
|     | reason               | TEXT         |                   | Explanation or reason for the decision                               |

```sql
CREATE TABLE IF NOT EXISTS authorization.t_policy_decision_logs
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    user_id UUID NOT NULL REFERENCES authentication.t_users(id),
    policy_id UUID REFERENCES authorization.t_policies(id),
    resource VARCHAR(256),
    action VARCHAR(128),
    decision VARCHAR(16) NOT NULL CHECK (decision IN ('ALLOW','DENY')),
    evaluated_attributes JSONB,
    reason TEXT
);
CREATE INDEX IF NOT EXISTS idx_t_policy_decision_logs_user_id_policy_id ON authorization.t_policy_decision_logs(user_id, policy_id);
```

### Table Constraints

| Constraint Type | Constraint Name                              | Description                                                                         |
| --------------- | -------------------------------------------- | ----------------------------------------------------------------------------------- |
| PRIMARY KEY     | id                                           | Defines `id` as the primary key                                                     |
| FOREIGN KEY     | created_by                                   | References `authentication.t_users(id)` for the creator                             |
| FOREIGN KEY     | user_id                                      | References `authentication.t_users(id)` for the user                                |
| FOREIGN KEY     | policy_id                                    | References `authorization.t_policies(id)` for the policy                            |
| CHECK           | decision                                     | Ensures `decision` is either 'ALLOW' or 'DENY'                                      |
| INDEX           | idx_t_policy_decision_logs_user_id_policy_id | Creates a composite index on `user_id` and `policy_id` to improve query performance |

## t_user_attribute_mappings

Table to store attribute values assigned to users, including metadata, status, and reference to attribute definitions.

### Table Columns

| Key | Column Name  | Data Type | Default           | Description                                                    |
| --- | ------------ | --------- | ----------------- | -------------------------------------------------------------- |
| PK  | id           | UUID      | GEN_RANDOM_UUID() | Primary key of the user-attribute mapping record               |
| FK  | created_by   | UUID      |                   | References the creator from `authentication.t_users(id)`       |
|     | created_at   | TIMESTAMP | CURRENT_TIMESTAMP | Timestamp when the record was created                          |
| FK  | updated_by   | UUID      |                   | References the last updater from `authentication.t_users(id)`  |
|     | updated_at   | TIMESTAMP |                   | Timestamp of the last update                                   |
|     | is_active    | BOOLEAN   | FALSE             | Active status of the mapping record                            |
| FK  | inactive_by  | UUID      |                   | References the user who deactivated this mapping               |
|     | inactive_at  | TIMESTAMP |                   | Timestamp when the mapping was deactivated                     |
|     | is_deleted   | BOOLEAN   | FALSE             | Logical deletion flag                                          |
| FK  | deleted_by   | UUID      |                   | References the user who deleted this mapping                   |
|     | deleted_at   | TIMESTAMP |                   | Timestamp when the mapping was deleted                         |
| FK  | user_id      | UUID      |                   | References the user from `authentication.t_users(id)`          |
| FK  | attribute_id | UUID      |                   | References the attribute from `authorization.m_attributes(id)` |
|     | value        | TEXT      |                   | Value assigned to the attribute for the user                   |

```sql
CREATE TABLE IF NOT EXISTS authorization.t_user_attribute_mappings
(
    id UUID NOT NULL DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
    created_by UUID NOT NULL REFERENCES authentication.t_users(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES authentication.t_users(id),
    updated_at TIMESTAMP,

    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    inactive_at TIMESTAMP,
    inactive_by UUID REFERENCES authentication.t_users(id),

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by UUID REFERENCES authentication.t_users(id),

    user_id UUID NOT NULL REFERENCES authentication.t_users(id),
    attribute_id UUID NOT NULL REFERENCES authorization.m_attributes(id),
    value TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_t_user_attribute_mappings_user_id_attribute_id ON authorization.t_user_attribute_mappings(user_id, attribute_id);
```

### Table Constraints

| Constraint Type | Constraint Name                                    | Description                                                                            |
| --------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------- |
| PRIMARY KEY     | id                                                 | Defines `id` as the primary key                                                        |
| FOREIGN KEY     | created_by                                         | References `authentication.t_users(id)` for the creator                                |
| FOREIGN KEY     | updated_by                                         | References `authentication.t_users(id)` for the last updater                           |
| FOREIGN KEY     | inactive_by                                        | References `authentication.t_users(id)` for deactivation                               |
| FOREIGN KEY     | deleted_by                                         | References `authentication.t_users(id)` for deletion                                   |
| FOREIGN KEY     | user_id                                            | References `authentication.t_users(id)` for the user                                   |
| FOREIGN KEY     | attribute_id                                       | References `authorization.m_attributes(id)` for the attribute                          |
| INDEX           | idx_t_user_attribute_mappings_user_id_attribute_id | Creates a composite index on `user_id` and `attribute_id` to improve query performance |
