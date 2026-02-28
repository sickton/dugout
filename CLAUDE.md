# Dugout — Project Context for Claude Code
*Hand this to Claude Code at the start of every session*

---

## What This Project Is

**Dugout** is a full-stack football management simulator designed to make football understandable and engaging for complete beginners. The user takes the role of a football manager for one simulated 90-minute match. They make real decisions in plain English when the game asks for them. The AI manages the opposing team. At full time, the product checks whether this fixture happened in real life in the last 5 years and compares the user's result to what actually happened.

**The core insight driving everything:**
A 0-0 scoreline looks like nothing happened. In reality it is often the most tactically rich result in football — two teams so intelligently organised that neither could break the other down. Dugout makes beginners feel that, not just understand it intellectually.

**The one sentence:**
> A football companion that puts complete beginners in the dugout — making real decisions, feeling real consequences, and understanding for the first time what actually happens across 90 minutes of football.

---

## The Target User

Someone with a slight idea about football. They know there are two teams, 11 players, and the goal is to score. They have watched matches before but switched off in indifference — not confusion. They are one good moment away from genuinely caring. They are not a football fan yet. They are not a complete beginner either.

---

## Tech Stack

```
Backend:     Java 21 + Spring Boot 4.0.3 (Maven)
ML:          Python 3.14 + scikit-learn, pandas, numpy
Frontend:    Vite + React (JavaScript)
Database:    PostgreSQL (local, database name: dugout)
Repo:        GitHub — sickton/dugout (public)
IDE:         IntelliJ IDEA Ultimate
```

---

## Project Structure

```
Dugout/                          ← root repo (cloned from GitHub)
├── README.md
├── LICENSE                      ← MIT
├── .gitignore
├── dugout-backend/              ← Spring Boot module
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/dugout/backend/
│       │   └── DugoutBackendApplication.java
│       └── resources/
│           └── application.properties
├── dugout-ml/                   ← Python ML module
│   ├── .venv/                   ← virtual environment
│   ├── data/
│   │   ├── raw/                 ← CSVs go here (gitignored)
│   │   └── sample/
│   ├── notebooks/               ← Jupyter notebooks
│   ├── models/                  ← trained model outputs
│   └── src/
└── dugout-frontend/             ← Vite React module
    ├── package.json
    └── src/
```

---

## Current application.properties

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/dugout
spring.datasource.username=postgres
spring.datasource.password=[user's actual password]
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.datasource.hikari.initialization-fail-timeout=-1
```

---

## What Is Currently Working

```
✓ Spring Boot backend starts and connects to PostgreSQL on port 8080
✓ Python venv active with pandas, numpy, scikit-learn, matplotlib, seaborn installed
✓ Vite React frontend runs on port 5173
✓ PostgreSQL running locally with dugout database created
✓ All three modules committed and pushed to GitHub
```

---

## The Simulation — How It Works

### Match Structure
```
Total match length = 90 + x + y
x = random integer 0-8   → added after minute 45 (first half extra time)
y = random integer 0-10  → added after minute 90 (second half extra time)
```

### Core Model
The simulation models two agents sharing one resource — the ball. At any moment exactly one team has possession. The ball moves through three zones:

```
Defensive Third → Midfield → Attacking Third
```

Players are drawn from zone-appropriate positions. A centre back does not appear as a chance-taker. A striker does not appear making a tackle in the defensive third unless specifically triggered by a set piece.

### Event Loop
For every event the engine asks:
1. Who has possession and in which zone?
2. Does play progress to the next zone or does possession switch?
3. If the ball reaches the attacking third — is a chance created?
4. Which player is involved? (weighted by individual ability score)
5. Does it result in a goal? (team strength + player ability + random roll)

### Decision Moments
Before kickoff, 5-7 specific minute numbers are generated and distributed across the match. When the simulation clock hits each flagged minute it pauses and presents a plain English scenario to the user. The user picks from 2-3 options. Their choice applies a probability modifier for the next 10-20 minutes of simulation time.

### User Decision Modifiers
```
Press them high     → user chance rate up, opponent chance rate up
Keep the ball       → possession retention up, chance rate slightly down
Sit deep            → opponent chance rate down, counter probability up
Push everyone forward → user chance rate sharply up, defensive vulnerability sharply up
```

### AI Opponent
Two layers:
- Team selection: AI picks a team that counters the user's choice (possession team vs counter-attacking side etc.)
- In-match adaptation: AI reads match state every N minutes and adjusts its modifier (if user presses high → AI goes long; if AI is losing late → AI increases aggression)

### The Killer Feature
After full time, the system checks whether this fixture happened in real life in the last 5 years. If it did, it shows:
- The real result
- What the real manager actually decided at key moments
- The one moment that decided the real match vs the user's simulation

---

## Player Data Model

Every player needs three ability dimensions:

```json
{
  "name": "Mohamed Salah",
  "team": "Liverpool",
  "role": "Winger",
  "zone": "Attacking Third",
  "ability": {
    "attacking": 92,
    "defending": 45,
    "passing": 81
  }
}
```

Role determines which ability score the simulation uses:
```
Striker      → attacking is primary
Centre Back  → defending is primary
Fullback     → all three matter
Central Mid  → passing is primary
Goalkeeper   → defending is everything
```

---

## ML Components

### 1. Player Role Classifier (CURRENT PRIORITY)
**What:** Classifies players into specific roles — Striker, Winger, Central Mid, Defensive Mid, Fullback, Centre Back, Goalkeeper
**Why:** The simulation needs to know which zone each player belongs to and which ability dimension applies
**Data:** Big 5 European leagues 2024-25 player stats from Kaggle
**Approach:** Supervised classification (Random Forest or similar) trained on player stats with FBref position codes as labels
**Output:** Role label + zone assignment per player

### 2. Goal Probability Model
**What:** Predicts probability of a goal given match context
**Why:** Calibrates simulation randomness with real football data instead of hand-tuned numbers
**Features:** Team attack vs defence rating differential, player ability score, zone, match minute, score state
**Approach:** Logistic regression or gradient boosting trained on historical match data
**Output:** Probability value 0-1 used in simulation goal resolution

### 3. Adaptive AI Opponent
**What:** AI that learns the user's decision patterns and adapts during a session
**Why:** A predictable rule-based AI becomes exploitable quickly
**Approach:** Inspired by reinforcement learning — tracks user decision history, identifies patterns, adjusts tactical modifier
**Output:** Dynamic AI modifier that changes based on user tendencies

### 4. Historical Fixture Similarity
**What:** Finds the most similar real fixture to the user's simulated match
**Why:** Powers the killer feature — "this is what actually happened"
**Features:** Teams, scoreline progression, chance counts, possession, momentum shifts
**Approach:** Cosine similarity or k-nearest neighbours on match feature vectors
**Output:** Closest real fixture from 5-year historical database

---

## REST API Design

```
POST /match/start
     → Takes user team selection
     → AI selects opponent
     → Generates match length and moment minutes
     → Runs simulation to first moment
     → Returns match state + first scenario

POST /match/decision
     → Takes user decision for current moment
     → Applies modifier
     → Continues simulation to next moment or full time
     → Returns updated match state + next scenario

GET  /match/state/{matchId}
     → Returns current match state

GET  /match/result/{matchId}
     → Returns final score, decision recap, learning summary

GET  /match/historical/{matchId}
     → Runs similarity model, returns closest real fixture

GET  /teams
     → Returns available teams with profiles
```

---

## Spring Boot Layer Architecture

```
MatchController      → HTTP requests/responses only, no business logic
MatchService         → orchestrates match lifecycle
SimulationEngine     → core event loop, possession, zones, probability
ScenarioService      → generates plain English situations for decision moments
AIOpponentService    → AI decision function, reads and adapts to match state
HistoricalService    → fixture similarity search, post-match comparison
TeamService          → loads team and player data
PlayerService        → loads and serves player profiles with ability scores
```

---

## Concurrency Design Decision

Every match gets a unique UUID as matchId. All state is stored per matchId. No static variables. No singleton simulation instances. No shared mutable state between matches. This means multiple concurrent users are supported by the architecture from day one — scaling at deployment is an infrastructure decision, not a code rewrite.

---

## What Has NOT Been Built Yet

```
→ Player role classifier (current priority — data collected, not yet trained)
→ Simulation engine Java implementation
→ REST API endpoints
→ React frontend UI
→ Goal probability ML model
→ Historical fixture database
→ PostgreSQL schema and entities
→ Adaptive AI opponent
→ Historical similarity model
```

---

## What To Build First — The Order

```
1. Player role classifier (dugout-ml)
   → Clean the Kaggle dataset
   → Engineer features
   → Train classifier
   → Output role-labelled player JSON

2. Player ability scoring (dugout-ml)
   → Calculate attacking, defending, passing scores per role
   → Export final player JSON

3. Spring Boot data models (dugout-backend)
   → Player entity, Team entity, MatchState entity
   → JPA repositories
   → Load player JSON into database

4. Simulation engine (dugout-backend)
   → Event loop
   → Possession and zone model
   → Goal probability calculation
   → Decision modifier system

5. REST API (dugout-backend)
   → All endpoints listed above

6. React frontend (dugout-frontend)
   → Team selection screen
   → Match ticker
   → Decision moment UI
   → Full time screen
   → Historical comparison screen
```

---

## Teams In Scope (Initial Set)

```
Premier League:  Liverpool, Arsenal, Manchester City,
                 Manchester United, Tottenham Hotspur, Chelsea
La Liga:         Barcelona, Real Madrid
Bundesliga:      Bayern Munich
Ligue 1:         PSG
```

---

## Key Design Rules — Never Break These

1. **No jargon ever reaches the user.** HIGH_PRESS, TIKI_TAKA, CONTROL — these live in the engine only. The user sees plain English always.

2. **Decisions shift probabilities, they do not determine outcomes.** A good decision increases the chance of a good result. It does not guarantee one. This mirrors real football.

3. **Every match state is isolated by matchId.** Never share state between matches.

4. **The AI feels like it is reading you.** It reacts to patterns, not just to the last decision.

5. **Wrong decisions teach, they do not punish.** The explanation after a bad choice is the most important moment in the experience.

---

*Project started: February 2026*
*Status: Foundation complete — three modules running, starting ML classifier next*
*GitHub: https://github.com/sickton/dugout*