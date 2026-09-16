# Q&A Platform — PostgreSQL DBMS Project

A database-driven Question & Answer platform built using **PostgreSQL and PL/pgSQL**, designed to model the core functionality of a collaborative discussion platform.

The project focuses on relational database design, data integrity, role-based access control, stored procedures, triggers, indexing, views, and automated database-side business logic.

---

## About the Project

The goal of this project was to design a relational database capable of supporting the core functionality of a Question & Answer platform where users can:

* Create accounts
* Ask questions
* Post answers
* Comment on posts
* Upvote or downvote content
* Associate tags with questions
* Track user and post scores
* Identify highly rated answers
* View leaderboard rankings
* Explore popular questions and tags

Rather than implementing most of the application logic externally, the project makes extensive use of **PostgreSQL procedures, functions, triggers, constraints, roles, and views** to enforce rules and maintain data consistency directly at the database layer.

---

## Key Features

### User Management

Users can create accounts and maintain profile information such as:

* Username
* Location
* User bio
* Reputation score
* Number of views
* Upvotes and downvotes
* Account creation and last-access timestamps

Each user can also be mapped to a PostgreSQL role to support database-level authentication and authorization.

---

### Questions and Answers

Posts are modeled using a common `posts` table.

A post can represent:

* **Question** — `post_type_id = 1`
* **Answer** — `post_type_id = 2`

Answers reference their corresponding question using a `parent_id`.

Questions maintain additional information such as:

* Number of answers
* Number of comments
* Score
* Upvotes
* Downvotes
* Tags
* Best answer

This design allows questions and answers to share a common structure while maintaining their relationship through self-referencing foreign keys.

---

### Voting and Reputation

Users can upvote or downvote posts.

When a vote is recorded, PostgreSQL triggers automatically update related information such as:

* Post score
* Number of upvotes
* Number of downvotes
* User reputation
* User voting statistics
* Best answer for a question

This keeps derived values synchronized without requiring the application layer to manually update multiple tables.

---

### Automatic Best Answer Selection

When votes are added to answer posts, the platform automatically evaluates the scores of all answers belonging to the same question.

The answer with the highest score is stored as the question's `best_answer_id`.

This logic is handled through a database trigger.

---

### Comment Management

Users can add comments to posts.

Triggers automatically maintain the `comment_count` associated with each post when comments are inserted or deleted.

---

### Tag Management

Questions can be associated with tags.

When a new post is created:

* If the tag already exists, its usage count is incremented.
* If the tag does not exist, a new tag entry is created.

When posts are removed, the corresponding tag count is updated automatically.

---

## Database Design

The platform is centered around the following core entities:

| Table      | Purpose                                                                   |
| ---------- | ------------------------------------------------------------------------- |
| `users`    | Stores user profiles, scores, voting statistics, and activity information |
| `posts`    | Stores both questions and answers                                         |
| `votes`    | Records upvotes and downvotes made by users                               |
| `comments` | Stores comments associated with posts                                     |
| `tags`     | Stores tags and their usage counts                                        |
| `badges`   | Stores badge classifications assigned to users                            |

The database uses:

* Primary keys
* Foreign keys
* Self-referencing relationships
* Default values
* Check constraints
* Role-based privileges
* Triggers
* Stored procedures
* PostgreSQL functions

to maintain relational integrity and automate application behavior.

---

## Database Relationships

Conceptually, the system follows relationships similar to:

```text
Users
  |
  | creates
  v
Posts -------------------+
  |                      |
  | receives             | parent_id
  v                      |
Votes                  Answers
  |
  +---- User

Posts
  |
  +---- Comments

Posts
  |
  +---- Tags

Users
  |
  +---- Badges
```

Questions and answers are represented in the same `posts` table, with answer posts referencing their parent question.

---

## Stored Procedures and Functions

A major part of the project is implemented directly using **PL/pgSQL stored procedures and functions**.

### `create_new_user`

Creates a new platform user and stores their profile information.

It also invokes the PostgreSQL role creation logic so that the user can be assigned appropriate database privileges.

---

### `create_user_role`

Creates a PostgreSQL login role using the format:

```text
user_<user_id>
```

The new role inherits permissions from the common `client_user` role.

---

### `create_new_post`

Creates either a question or an answer.

The procedure accepts information such as:

* Post type
* Post body
* Parent question ID
* Tag

For answer posts, `parent_id` links the answer to its corresponding question.

---

### `create_new_comment`

Adds a comment to a post while associating it with the currently authenticated user.

---

### `create_vote`

Records an upvote or downvote for a post.

Vote insertion subsequently activates triggers responsible for updating post scores, user reputation, and related statistics.

---

### `delete_post`

Deletes a post.

Deletion is protected by database-side authorization logic so that regular users cannot delete posts belonging to other users.

---

### `get_ans_of_post`

Returns all answers belonging to a given question.

---

### `get_user_info`

Retrieves information for a specific user.

---

### `get_tag_posts`

Returns posts associated with a particular tag.

---

### `get_comm_of_post`

Returns all comments belonging to a given post.

---

### `update_badge_class`

Updates badge classifications based on user scores.

The operation is restricted to the `leaderboard_manager` role.

---

### `ban_users`

Allows the moderator role to identify users whose score falls below the configured threshold.

Such accounts are marked for removal.

---

## Triggers

The project uses triggers extensively to maintain consistency between related database records.

### `del_post`

Ensures that normal users can only delete posts that they created.

---

### `insert_badge`

Automatically assigns a default badge entry when a new user is created.

---

### `best_ans_upd`

Executed when a vote is inserted.

It automatically updates:

* Post score
* Post upvotes/downvotes
* User score
* User voting statistics
* Best answer for the corresponding question

---

### `upd_ans_ct`

Automatically increments a question's `answer_count` when a new answer is created.

---

### `upd_ans_ct_del`

Decrements the answer count when an answer is deleted and also updates the associated tag count.

---

### `upd_tag_ct`

Maintains tag usage statistics whenever a new tagged post is inserted.

---

### `upd_comm_ct`

Increments the comment count when a new comment is added.

---

### `upd_del_comm_ct`

Decrements the comment count when a comment is deleted.

---

## Role-Based Access Control

The database implements different roles with different privileges.

### `client_user`

Represents regular users of the platform.

Users receive permissions required for normal platform operations such as:

* Viewing records
* Creating posts
* Updating permitted data
* Adding comments
* Voting
* Working with tags

Individual users are represented using PostgreSQL roles such as:

```text
user_123
user_456
```

These roles inherit privileges from `client_user`.

---

### `managers`

Base role used for administrative functionality.

---

### `moderator`

Responsible for moderation-related operations.

The moderator can perform privileged operations such as managing users and posts and identifying accounts that violate reputation rules.

---

### `leaderboard_manager`

Responsible for maintaining badge and leaderboard-related information.

This separates platform responsibilities and demonstrates **Role-Based Access Control (RBAC)** at the database level.

---

## Views

The project defines several views for commonly accessed information.

### `leader_board`

Provides information about the highest-ranked users based on their scores.

The view is designed to expose the top users in the platform leaderboard.

---

### `featured_questions`

Contains highly rated question posts.

This provides an easy way to surface popular discussions without repeatedly writing the underlying ranking query.

---

### `top5_tags`

Returns the most frequently used tags across the platform.

This helps identify the topics most commonly discussed by users.

---

## Indexing and Query Optimization

Indexes were added to frequently accessed attributes to improve query performance.

The project includes hash indexes on identifiers such as:

```sql
users(user_id)
posts(post_id)
votes(vote_id)
```

and an index on:

```sql
users(score)
```

These indexes support frequent operations including:

* User lookup
* Post retrieval
* Post deletion
* Answer retrieval
* Vote processing
* Leaderboard and moderation operations

---

## Database-Side Business Logic

One of the main design decisions in this project was to place important consistency rules inside the database.

For example, adding a vote may affect several pieces of data:

```text
Vote inserted
      |
      v
Update post score
      |
      +--> Update upvote/downvote count
      |
      +--> Update author's reputation
      |
      +--> Update user's voting statistics
      |
      +--> Recalculate best answer
```

Instead of requiring every application consuming the database to implement this logic correctly, triggers centralize these rules inside PostgreSQL.

---

## Simple Frontend Integration

The repository also contains a lightweight frontend prototype using:

* **Python**
* **Streamlit**
* **psycopg2**

The application demonstrates connecting to PostgreSQL using user credentials and retrieving database records.

The main focus of the project, however, is the **database architecture and DBMS implementation**, rather than frontend development.

---

## Tech Stack

### Database

* PostgreSQL
* SQL
* PL/pgSQL

### Application

* Python
* Streamlit
* psycopg2

### Database Concepts

* Relational Database Design
* Database Normalization
* Primary and Foreign Keys
* Self-Referencing Relationships
* Integrity Constraints
* Stored Procedures
* Functions
* Triggers
* Views
* Indexing
* Query Optimization
* Role-Based Access Control
* Database Authorization

---

## Project Structure

```text
Q-A-Platform/
│
├── QA_portal.sql
│   └── Main SQL implementation containing roles,
│       procedures, triggers, views, indexes, and queries
│
├── functions.sql
│   └── Stored procedures and utility functions
│
├── triggers.sql
│   └── Database triggers and trigger functions
│
├── project/
│   ├── main.py
│   ├── main2.py
│   └── requirements.txt
│
├── DBMS Final Project Report.pdf
│   └── Database design, ER model, constraints,
│       functions, triggers, roles, indexes, and queries
│
└── README.md
```

---

## Running the Project

### Prerequisites

Install:

* PostgreSQL
* Python 3.x

For the Python prototype:

```bash
pip install psycopg2-binary streamlit
```

---

### Database Setup

Create a PostgreSQL database and execute the required database schema and SQL scripts.

The repository contains SQL definitions for:

* User roles and privileges
* Stored procedures
* Functions
* Triggers
* Views
* Indexes

Because the scripts contain role-management and privilege-management operations, some commands may require a PostgreSQL account with sufficient privileges.

---

### Run the Streamlit Prototype

After configuring the PostgreSQL connection:

```bash
streamlit run project/main2.py
```

The frontend can authenticate using PostgreSQL credentials and retrieve data from the database.

---

## What I Learned

This project helped strengthen my understanding of how database systems can enforce application rules beyond simply storing data.

Some of the main concepts explored were:

* Designing relational schemas for interconnected entities
* Modeling questions and answers using self-referencing relationships
* Using triggers to keep derived data synchronized
* Writing stored procedures for reusable database operations
* Applying database-level authentication and authorization
* Using roles and privileges to restrict operations
* Maintaining data integrity through constraints
* Using indexes and views to improve common database operations
* Connecting a Python application to PostgreSQL

It also demonstrated how business logic can be distributed between the application and database layers depending on consistency, security, and maintainability requirements.

---

## Possible Improvements

The project was developed primarily to explore DBMS concepts, so there are several directions in which it could be extended:

* Build a complete REST API around the database
* Replace direct SQL construction in the prototype with parameterized queries
* Store credentials using environment variables rather than source code
* Improve authentication and session management
* Add transaction handling for multi-step operations
* Add database migrations
* Add automated database integration tests
* Implement pagination and full-text search
* Support multiple tags per question through a many-to-many relationship
* Add richer moderation workflows
* Containerize PostgreSQL and the backend using Docker
* Build a production-style backend using FastAPI or another framework

These extensions could turn the academic DBMS implementation into a more complete backend application.

---

## Project Context

This project was developed as an academic **Database Management Systems project** to apply relational database concepts to a realistic Question & Answer platform.

The emphasis was on understanding how database design, constraints, triggers, procedures, privileges, indexing, and views can work together to create a consistent and structured application data layer.

