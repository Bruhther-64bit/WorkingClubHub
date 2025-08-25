# ClubHub — Best Platform for University Clubs

A role‑based Spring Boot web app that helps students discover, follow and join university clubs, club admins manage communities, and university admins oversee clubs — all in one place.

- Stack: Spring Boot 3, Spring Security 6, Spring Data JPA (Hibernate), PostgreSQL, Flyway, Thymeleaf, CSS/JS design system
- Java: 21
- Build: Maven

## Features

- Authentication & roles
  - Email/password login with role-based access: STUDENT, CLUB_ADMIN, UNIVERSITY_ADMIN
  - Clean UX guards: no browser basic-auth popups; friendly redirects/modals for unauthorized actions
- Explore clubs
  - Reddit-style club cards (gradient banners, avatar overlap), Follow/Unfollow, status badges
  - Search, filters (country/university), sorts (name/followers/members/recent)
- Student
  - Narrow, Facebook-like feed from followed and joined clubs (stats-only preview)
  - Post creation (text + optional image) for clubs they’re members of
  - University-specific clubs page with Apply/Cancel/Reapply and Follow toggles
  - Notifications for application decisions (read/unread)
- Club Admin
  - Admin feed with image thumbnails
  - Post create/edit/delete, replace/remove images
  - Member list + remove, Applications review (Accept/Reject with notification)
  - Comment moderation on club posts
- University Admin
  - Create a club + create the club admin in one transaction
  - Club list and Manage admin page (prefilled; passwords blank by default)
- Media uploads
  - Local disk storage (configurable), served from `/uploads/**` via Spring MVC resource handler
- Design system
  - Consolidated `site.css` and `fragments/common.html` (role-aware nav, theme toggle, unread badge)
  - Page-scoped CSS/JS only when needed

## Quick start

### Prerequisites
- Java 21+
- Maven 3.9+
- PostgreSQL 14+ (local or container)

### 1) Clone and configure
```bash
git clone https://github.com/your-org/clubhub.git
cd clubhub
```

Create a database and user (example):
```sql
CREATE DATABASE clubhub;
CREATE USER clubhub_user WITH PASSWORD 'clubhub_pass';
GRANT ALL PRIVILEGES ON DATABASE clubhub TO clubhub_user;
```

### 2) application.properties
Create `src/main/resources/application.properties`:
```properties
# Database
spring.datasource.url=jdbc:postgresql://localhost:5432/clubhub
spring.datasource.username=clubhub_user
spring.datasource.password=clubhub_pass

# JPA
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true

# Flyway
spring.flyway.enabled=true

# Static uploads (relative to project root; portable)
app.media.upload-dir=uploads

# Multipart limits
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=6MB

# Thymeleaf
spring.thymeleaf.cache=false
```

### 3) Run
```bash
./mvnw spring-boot:run
# or
mvn spring-boot:run
```

Open http://localhost:8080

### 4) Demo logins
Use these on the Login page:
- Student: sophia.walker.62@students.u005.edu / s162
- Club Admin: ca1_1@clubs.u001.edu / ca1
- University Admin: jordan.williams.ua1@admin.u001.edu / ua1

If you’re seeding your own DB, adapt these to your dataset.

## How it’s organized

```
src/main/java/com/example/clubhub4
  ├─ config/              # SecurityConfig, WebMvcConfig (serves /uploads/**)
  ├─ controller/          # MVC controllers (Explore, PublicClub, Student, ClubAdmin, UnivAdmin, Auth, Notifications)
  ├─ entity/              # JPA entities: User, Club, Post, Follow, ClubMember, ClubApplication, Notification, ...
  ├─ repository/          # Spring Data JPA repositories
  ├─ security/            # AppUserPrincipal, etc.
  ├─ explore/, feed/, ... # Services and view models (CardView, PostView, etc.)
  └─ university/          # Univ admin forms and services

src/main/resources/
  ├─ templates/           # Thymeleaf pages
  │  ├─ fragments/common.html
  │  ├─ club/, clubadmin/, student/, explore/, university/, auth pages
  ├─ static/css/site.css  # consolidated design system
  ├─ static/js/site.js    # global interactions (theme, toasts, badge, grid sizing)
  └─ db/migration/        # Flyway SQL migrations
```

## Security & roles

- Public: `/`, `/dashboard`, `/explore/**`, `/clubs/**`, `/login`, `/signup`, `/uploads/**`
- Student (ROLE_STUDENT): `/student/**`, `/me/posts/**`
- Club Admin (ROLE_CLUB_ADMIN): `/club/**`
- University Admin (ROLE_UNIVERSITY_ADMIN): `/university/**`
- Actions (auth required): Follow, Like, Comment, Apply, Member posting, etc.

Form login: `/login` (email + password). CSRF is enabled and included in all forms.

### Important note on passwords
For local/dev, the project uses `NoOpPasswordEncoder` (plain) unless you switch it. For production:

1) Replace the encoder in SecurityConfig:
```java
@Bean
public PasswordEncoder passwordEncoder() {
  return new BCryptPasswordEncoder();
}
```
2) Store BCrypt-hashed passwords in `users.password_hash`.

## Database & migrations

Core tables include:
- `users` (STUDENT | CLUB_ADMIN | UNIVERSITY_ADMIN)
- `university`, `country`
- `club` (unique per university), `clubmember`, `follow`
- `post`, `postlikes`, `postcomment`
- `clubapplication` (PENDING/ACCEPTED/REJECTED), `notification`

Flyway runs migrations at startup from `src/main/resources/db/migration`. Include:
- Extensions (e.g., `uuid-ossp` if your SQL defaults use it)
- Tables, constraints (e.g., `unique_club_admin` on `club.admin_id`)
- Indexes (performance): `follow(user_id, club_id)`, `clubmember(user_id, club_id)`,
  `post(club_id, created_at)`, `postlikes(post_id)`, `postcomment(post_id, created_at)`

## Media uploads

- Configure the target folder with `app.media.upload-dir` (default `uploads` in project root).
- Files are served from `/uploads/**` via `WebMvcConfig`.
- The Post entity has `imageUrl` that stores the web path (e.g., `/uploads/uuid.jpg`).
- For portability, keep the `uploads/` folder in your repo/zip (use a `.keep` file if empty).

## Notable interactions

- Explore and University Clubs pages:
  - Follow/Unfollow, Apply/Cancel/Reapply with robust form behavior (no click-through on the card).
  - Button text reflects state via `followedIds` and application maps recalculated on redirect.
- Student feed:
  - Narrow page width + image height capping for a clean, readable stream.
- Guards:
  - Unauthorized tile clicks show a friendly modal (if logged in) or redirect to `/login`
    (no basic-auth popups, no whitelabel errors).

## Running Postgres in Docker (optional)

```yaml
# docker-compose.yml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: clubhub
      POSTGRES_USER: clubhub_user
      POSTGRES_PASSWORD: clubhub_pass
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
volumes:
  pg_data:
```

Then update `spring.datasource.url` to `jdbc:postgresql://localhost:5432/clubhub`.

## Development tips

- Cache-bust CSS/JS via versions in `fragments/common.html`: `?v=5`
- Toggle light/dark with the theme button (persists in localStorage)
- Enable logs when debugging:
```properties
logging.level.org.springframework.security=INFO
logging.level.org.hibernate.SQL=DEBUG
```

## Roadmap ideas

- Async toggles (Follow/Like) to avoid full page reloads
- Profile settings (change university, profile picture)
- Email verification / password reset
- REST API for mobile/SPAs

## License

MIT (or your preferred license). See `LICENSE`.

---

If you run into issues getting started (e.g., button text not toggling, null model attributes in templates, or DB constraints), open an issue with the stack trace and the page you were on — the README links above reflect how the app is wired end-to-end. Happy hacking!
