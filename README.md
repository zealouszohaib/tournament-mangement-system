
# ⚽ PostgreSQL Database Schema (8 Tables)

```sql
-- 1) Clubs
CREATE TABLE clubs (
    club_id SERIAL PRIMARY KEY,
    club_name VARCHAR(150) NOT NULL UNIQUE,
    address VARCHAR(255),
    contact VARCHAR(50),
    email VARCHAR(150),
    founded_year INT
);

-- 2) Sports
CREATE TABLE sports (
    sport_id SERIAL PRIMARY KEY,
    club_id INT NOT NULL REFERENCES clubs(club_id) ON DELETE CASCADE,
    sport_name VARCHAR(100) NOT NULL,
    rules_description TEXT,
    CONSTRAINT uq_sport_per_club UNIQUE (club_id, sport_name)
);

-- 3) Members (players, coaches, referees, admins)
CREATE TABLE members (
    member_id SERIAL PRIMARY KEY,
    club_id INT NOT NULL REFERENCES clubs(club_id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL CHECK (role IN ('Player', 'Coach', 'Referee', 'Admin')),
    first_name VARCHAR(60) NOT NULL,
    last_name VARCHAR(60),
    dob DATE,
    gender VARCHAR(10),
    phone VARCHAR(30),
    email VARCHAR(150),
    join_date DATE DEFAULT CURRENT_DATE,
    active BOOLEAN DEFAULT TRUE
);

-- 4) Teams
CREATE TABLE teams (
    team_id SERIAL PRIMARY KEY,
    sport_id INT NOT NULL REFERENCES sports(sport_id) ON DELETE CASCADE,
    team_name VARCHAR(120) NOT NULL,
    coach_member_id INT REFERENCES members(member_id) ON DELETE SET NULL,
    created_date DATE DEFAULT CURRENT_DATE,
    CONSTRAINT uq_team_per_sport UNIQUE (sport_id, team_name)
);

-- 5) TeamPlayers (many-to-many bridge: members <-> teams)
CREATE TABLE team_players (
    team_player_id SERIAL PRIMARY KEY,
    team_id INT NOT NULL REFERENCES teams(team_id) ON DELETE CASCADE,
    member_id INT NOT NULL REFERENCES members(member_id) ON DELETE CASCADE,
    position VARCHAR(50),
    jersey_number INT,
    joined_date DATE DEFAULT CURRENT_DATE,
    active BOOLEAN DEFAULT TRUE,
    CONSTRAINT uq_team_jersey UNIQUE (team_id, jersey_number)
);

-- 6) Tournaments
CREATE TABLE tournaments (
    tournament_id SERIAL PRIMARY KEY,
    sport_id INT NOT NULL REFERENCES sports(sport_id) ON DELETE CASCADE,
    tournament_name VARCHAR(150) NOT NULL,
    start_date DATE,
    end_date DATE,
    type VARCHAR(50),  -- e.g., 'Knockout','League','Round-robin'
    status VARCHAR(20) DEFAULT 'Planned'  -- 'Planned','Ongoing','Completed','Cancelled'
);

-- 7) Venues
CREATE TABLE venues (
    venue_id SERIAL PRIMARY KEY,
    club_id INT NOT NULL REFERENCES clubs(club_id) ON DELETE CASCADE,
    venue_name VARCHAR(150) NOT NULL,
    location VARCHAR(255),
    capacity INT,
    type VARCHAR(50),  -- e.g., 'Indoor','Outdoor'
    availability_status VARCHAR(20) DEFAULT 'Available',  -- 'Available','Booked','Maintenance'
    CONSTRAINT uq_venue_per_club UNIQUE (club_id, venue_name)
);

-- 8) Matches
CREATE TABLE matches (
    match_id SERIAL PRIMARY KEY,
    tournament_id INT NOT NULL REFERENCES tournaments(tournament_id) ON DELETE CASCADE,
    venue_id INT NOT NULL REFERENCES venues(venue_id) ON DELETE CASCADE,
    team1_id INT NOT NULL REFERENCES teams(team_id) ON DELETE CASCADE,
    team2_id INT NOT NULL REFERENCES teams(team_id) ON DELETE CASCADE,
    referee_member_id INT REFERENCES members(member_id) ON DELETE SET NULL,
    scheduled_at TIMESTAMP NOT NULL,
    status VARCHAR(20) DEFAULT 'Scheduled',  -- 'Scheduled','Finished','Postponed','Cancelled'
    score_team1 INT,
    score_team2 INT,
    result VARCHAR(100),
    CONSTRAINT chk_different_teams CHECK (team1_id <> team2_id),
    CONSTRAINT uq_match_slot UNIQUE (tournament_id, venue_id, scheduled_at)
);

-- ✅ Useful Indexes
CREATE INDEX idx_members_club ON members(club_id);
CREATE INDEX idx_teams_sport ON teams(sport_id);
CREATE INDEX idx_matches_tournament ON matches(tournament_id, scheduled_at);
CREATE INDEX idx_teamplayers_team ON team_players(team_id);
```

---

# ✅ Explanation (PostgreSQL Best Practices)

| Concept                | PostgreSQL Implementation                                                                                              |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **Auto-increment IDs** | `SERIAL` columns generate sequences automatically.                                                                     |
| **Dates & Times**      | Used `DATE` and `TIMESTAMP` with `CURRENT_DATE` or explicit inserts.                                                   |
| **Foreign keys**       | All `REFERENCES` use `ON DELETE CASCADE` to maintain clean relational integrity.                                       |
| **Booleans**           | `BOOLEAN` type replaces SQL Server’s `BIT`.                                                                            |
| **Text fields**        | Long text fields use `TEXT` instead of `VARCHAR(MAX)`.                                                                 |
| **Uniqueness rules**   | Multiple `UNIQUE` constraints for natural keys (e.g., `(club_id, sport_name)` and `(sport_id, team_name)`).            |
| **Enums**              | Instead of separate `enum` types, using `CHECK` constraints for roles and statuses keeps schema simpler for iteration. |

---

# 🧠 ERD Summary (Textual Overview)

```
Clubs (1) ────< (N) Sports
   │
   ├──< Members (1:N)
   │
   ├──< Venues (1:N)
   │
   └──< Teams (through Sports)

Sports (1) ────< (N) Teams
   │
   └──< Tournaments (1:N)
           │
           └──< Matches (1:N)
                    ├── Team1 (Teams)
                    ├── Team2 (Teams)
                    └── Referee (Members)

Teams (1) ────< (N) TeamPlayers (join table)
Members (1) ────< (N) TeamPlayers
```

---

# ⚙️ Example Data (for testing)

```sql
-- Clubs
INSERT INTO clubs (club_name, address, contact, email, founded_year)
VALUES ('Eagle Sports Club', 'Main Street 22', '0300-1234567', 'info@eagleclub.com', 2010);

-- Sports
INSERT INTO sports (club_id, sport_name) VALUES (1, 'Football'), (1, 'Cricket');

-- Members
INSERT INTO members (club_id, role, first_name, last_name, gender, email)
VALUES 
(1, 'Coach', 'Ali', 'Khan', 'Male', 'ali@club.com'),
(1, 'Player', 'Bilal', 'Ahmed', 'Male', 'bilal@club.com'),
(1, 'Player', 'Hassan', 'Raza', 'Male', 'hassan@club.com'),
(1, 'Referee', 'Sara', 'Iqbal', 'Female', 'sara@club.com');

-- Teams
INSERT INTO teams (sport_id, team_name, coach_member_id)
VALUES (1, 'Eagle Warriors', 1);

-- TeamPlayers
INSERT INTO team_players (team_id, member_id, position, jersey_number)
VALUES (1, 2, 'Forward', 9), (1, 3, 'Midfielder', 7);

-- Tournaments
INSERT INTO tournaments (sport_id, tournament_name, start_date, end_date, type)
VALUES (1, 'Spring Cup 2025', '2025-03-01', '2025-03-15', 'Knockout');

-- Venues
INSERT INTO venues (club_id, venue_name, location, capacity, type)
VALUES (1, 'Eagle Stadium', 'Downtown', 5000, 'Outdoor');

-- Matches
INSERT INTO matches (tournament_id, venue_id, team1_id, team2_id, referee_member_id, scheduled_at)
VALUES (1, 1, 1, 1, 4, '2025-03-02 16:00');  -- Will fail because of CHECK (same teams)
```

---

# 🚀 Summary

✅ **Compact (8 tables)**
✅ **Normalized (3NF)**
✅ **Covers all major tournament flows:**

* Clubs manage Sports, Venues, Members
* Coaches manage Teams
* Players join Teams
* Tournaments manage Matches
  ✅ **PostgreSQL-optimized types, constraints, and cleanup**

---
