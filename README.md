# Caro Game – Server

The **server** half of a multiplayer **Caro (Gomoku / Five-in-a-row)** game written in Java.
It accepts connections from game clients over TCP, handles accounts, rooms, friends, rankings and chat, and stores everything in a **MySQL** database. It also ships with a small **Admin panel** (desktop window) for broadcasting announcements, warning players and banning/unbanning them.

> 👉 You also need the client application to actually play: **Caro_Client** (https://github.com/NhuNguyen0101/Caro-Client).
> The server must be running **before** any client is started.

---

## Table of contents

1. [Features](#1-features)
2. [Tech stack](#2-tech-stack)
3. [What you need to install](#3-what-you-need-to-install-prerequisites)
4. [Step-by-step setup](#4-step-by-step-setup)
5. [Running the server](#5-running-the-server)
6. [Using the Admin panel](#6-using-the-admin-panel)
7. [Playing over a network (LAN / internet)](#7-playing-over-a-network-lan--internet)
8. [Project structure](#8-project-structure)
9. [Database schema](#9-database-schema)
10. [Network protocol reference](#10-network-protocol-reference)
11. [Troubleshooting](#11-troubleshooting)
12. [Known limitations](#12-known-limitations)

---

## 1. Features

- **TCP socket server** listening on port **7777**, one thread per connected client.
- **Accounts**: register (username, password, nickname, avatar) and log in. Duplicate usernames are rejected; one account cannot be logged in twice at the same time.
- **Rooms**: create a room (optionally password-protected), join by room ID, list open rooms, or "quick match" into any public room that is waiting for an opponent. Room IDs start at `100`.
- **Gameplay relay**: moves, in-game chat, draw offers, win/lose/time-out and "leave room" events are forwarded between the two players. The server also updates each player's game / win / draw counters.
- **Friends**: send a friend request, accept it, see your friend list with online/playing status, and challenge a friend to a duel.
- **Leaderboard**: top 8 players. Score formula used for ranking: `games played + draws × 5 + wins × 10`.
- **Global chat** between all online players.
- **Admin panel**: view connected threads and rooms, broadcast a server notice, warn a user, ban / unban a user by ID.

## 2. Tech stack

| Item | Details |
|------|---------|
| Language | Java (project is configured for **Java 15**; newer JDKs such as 17 or 21 work too) |
| UI (admin panel) | Java Swing |
| Networking | `java.net.ServerSocket` / `Socket` (plain-text, line based protocol over TCP) |
| Database | MySQL (via JDBC driver `mysql-connector-java-8.0.13.jar`, already included in `lib/`) |
| Build | NetBeans / Apache Ant project (`build.xml`, `nbproject/`) – but you can also build with plain `javac` |

## 3. What you need to install (prerequisites)

You need **two things** installed on your computer. Everything else (the MySQL JDBC driver) is already included in the `lib/` folder.

### 3.1 Java Development Kit (JDK) – version 15 or newer
1. Download a JDK from one of these (any is fine): [Adoptium Temurin](https://adoptium.net/) or [Oracle JDK](https://www.oracle.com/java/technologies/downloads/). **JDK 17 or 21 (LTS)** is recommended.
2. Install it.
3. Check it works – open a terminal (Windows: *Command Prompt* or *PowerShell*) and type:
   ```bash
   java -version
   javac -version
   ```
   Both commands must print a version number (15 or higher). If `javac` is "not recognized", the JDK's `bin` folder is not on your `PATH` – reinstall and tick the "Set JAVA_HOME / add to PATH" option, or add it manually.

### 3.2 MySQL Server (8.x recommended; 5.7 also works)
Pick **one** of these options:
- **MySQL Community Server + MySQL Workbench**: <https://dev.mysql.com/downloads/mysql/>
- **XAMPP** (easiest for beginners – includes MySQL/MariaDB and phpMyAdmin): <https://www.apachefriends.org/>

After installing, make sure the MySQL service is **running** (in XAMPP click *Start* next to *MySQL*). The default port is **3306**.

### 3.3 (Optional) An IDE
- **Apache NetBeans** (<https://netbeans.apache.org/>) opens this project directly because it is a NetBeans project. This is the easiest way to run it, but it is **not required**.
- IntelliJ IDEA / Eclipse / VS Code also work (create a project from existing sources and add `lib/mysql-connector-java-8.0.13.jar` to the classpath).

## 4. Step-by-step setup

### Step 1 – Get the code
```bash
git clone <your-server-repository-url>
cd Caro_Server-main
```
(or download the ZIP from GitHub and extract it).

### Step 2 – Create the database
The SQL script `src/sql/creat_db.sql` creates the **tables** but **not the database itself**, so create the database first.

**Option A – command line**
```bash
mysql -u root -p
```
then, inside the MySQL prompt:
```sql
CREATE DATABASE caro_game CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE caro_game;
SOURCE src/sql/creat_db.sql;
SHOW TABLES;
```
You should see three tables: `user`, `friend`, `banned_user`.
(If `SOURCE` cannot find the file, use the full path, e.g. `SOURCE C:/path/to/Caro_Server-main/src/sql/creat_db.sql;`)

**Option B – MySQL Workbench / phpMyAdmin**
1. Create a new schema/database named **`caro_game`**.
2. Select it, open `src/sql/creat_db.sql`, and run the whole script.

### Step 3 – Tell the server how to log in to your database
Open **`src/dao/DAO.java`** and edit these four values to match your MySQL setup:

```java
final String DATABASE_NAME = "caro_game"; // name of the database you created
final String jdbcURL = "jdbc:mysql://localhost:3306/" + DATABASE_NAME + "?useSSL=false";
final String JDBC_USER = "root";          // your MySQL username
final String JDBC_PASSWORD = "";          // your MySQL password (XAMPP default: empty)
```

- If MySQL runs on another machine or port, change `localhost:3306` accordingly.
- **Do not commit your real password to GitHub.** Before pushing, change it back to an empty string or load it from an environment variable.

### Step 4 – Build and run
See the next section.

## 5. Running the server

### Option 1 – NetBeans (easiest)
1. *File → Open Project…* and select the `Caro_Server-main` folder.
2. If NetBeans warns about a missing library `mssql-jdbc-9.2.1.jre15.jar`, you can safely **ignore** it or remove it (*Right-click project → Properties → Libraries*). It is not used by the code.
3. Right-click the project → **Run**. If asked for a main class, choose **`controller.Server`**.

### Option 2 – Command line (no IDE)
Run these commands **from inside the `Caro_Server-main` folder**.

**Windows (Command Prompt):**
```bat
mkdir build\classes
dir /s /b src\*.java > sources.txt
javac -encoding UTF-8 -cp lib\mysql-connector-java-8.0.13.jar -d build\classes @sources.txt
java -cp "build\classes;lib\mysql-connector-java-8.0.13.jar" controller.Server
```

**Linux / macOS:**
```bash
mkdir -p build/classes
javac -encoding UTF-8 -cp lib/mysql-connector-java-8.0.13.jar -d build/classes $(find src -name "*.java")
java -cp "build/classes:lib/mysql-connector-java-8.0.13.jar" controller.Server
```

> ⚠️ The `-encoding UTF-8` flag is important – the source code contains Vietnamese text, which will be garbled without it.

### What you should see
- The terminal prints `Server is waiting to accept user...`
- The **Admin panel** window opens.
- Each time a client connects you will see its IP address and `Server thread number N Started`.

If the console prints `Connection to database failed` followed by a stack trace, go to [Troubleshooting](#11-troubleshooting).

To stop the server, close the terminal / press the Stop button in your IDE (closing only the Admin window does not stop the server).

## 6. Using the Admin panel

The Admin window opens automatically when the server starts.

| Control | What it does |
|---------|--------------|
| **Xem danh sách luồng** ("View thread list") | Lists every connected client thread: client number, user ID (or `null` if not logged in) and current room. |
| **Xem danh sách phòng** ("View room list") | Lists all active rooms with their ID, number of players and the user IDs inside. |
| Message box + **Phát thông báo** ("Broadcast") | Sends a message to every online player; it appears in their global chat as *"Thông báo từ máy chủ : …"*. |
| User ID box + reason drop-down + **Cảnh cáo** ("Warn") | Shows a warning pop-up to that online user. |
| User ID box + reason drop-down + **Ban** | Adds the user to the `banned_user` table, kicks them back to the login screen and ends their current room. Banned users cannot log in. |
| User ID box + **Huỷ Ban** ("Unban") | Removes the ban so the user can log in again. |
| Log area | Shows who comes online/offline and global chat messages. |

Notes:
- User IDs are the `ID` column of the `user` table.
- Banning a user who is **not currently online** still saves the ban in the database, but the panel shows "Có lỗi xảy ra" (an error) because there is no live connection to notify.

## 7. Playing over a network (LAN / internet)

- The server listens on **TCP port 7777** on all network interfaces.
- The **client** currently has the server address hard-coded as `127.0.0.1` (same computer). To let other computers connect, each client's `SocketHandle.java` must be changed to the server's IP – see the client repository's README.
- Allow **TCP 7777** through the server computer's firewall (Windows: *Windows Defender Firewall → Advanced settings → Inbound Rules → New Rule → Port → TCP 7777*).
- On the same Wi-Fi/LAN use the server's local IP (find it with `ipconfig` on Windows or `ip a` / `ifconfig` on Linux/macOS, e.g. `192.168.1.20`).
- Over the internet you would additionally need port forwarding on your router (forward TCP 7777 to the server machine) and use your public IP. **Warning:** the protocol is unencrypted and passwords are stored in plain text, so do not expose this to the public internet with real passwords.

## 8. Project structure

```
Caro_Server-main/
├── build.xml                 # Ant build file (NetBeans)
├── manifest.mf
├── lib/
│   ├── mysql-connector-java-8.0.13.jar   # MySQL JDBC driver (required at runtime)
│   └── CopyLibs/             # NetBeans helper used when building a distributable JAR
├── nbproject/                # NetBeans project settings
└── src/
    ├── controller/
    │   ├── Server.java           # Entry point (main). Opens port 7777, starts Admin panel, accepts clients
    │   ├── ServerThread.java     # One per client: reads commands and replies (the whole protocol lives here)
    │   ├── ServerThreadBus.java  # List of all connected client threads; broadcast / send-to-user helpers
    │   └── Room.java             # A game room (2 players, optional password) + stat-updating helpers
    ├── dao/
    │   ├── DAO.java              # Opens the MySQL connection (edit your DB credentials here)
    │   └── UserDAO.java          # All SQL queries (login, register, friends, ranking, ban, stats…)
    ├── model/
    │   └── User.java             # Player data object
    ├── sql/
    │   └── creat_db.sql          # Creates the tables
    └── view/
        └── Admin.java            # Admin Swing window
```

## 9. Database schema

Created by `src/sql/creat_db.sql` in the database `caro_game`:

| Table | Columns | Purpose |
|-------|---------|---------|
| `user` | `ID` (PK, auto-increment), `username` (unique), `password`, `nickname`, `avatar`, `numberOfGame`, `numberOfWin`, `numberOfDraw`, `IsOnline`, `IsPlaying` | Player accounts and statistics |
| `friend` | `ID_User1`, `ID_User2` (composite PK, both FK → `user.ID`) | Friendships (stored once per pair) |
| `BANNED_USER` | `ID_User` (PK, FK → `user.ID`) | Banned players |

> **Linux users:** MySQL table names are case-sensitive on Linux by default, but the code mixes cases (for example it queries `banned_user` while the script creates `BANNED_USER`, and some queries use `User.ID` / `Friend.ID_User1`). If you see errors like `Table 'caro_game.banned_user' doesn't exist` or `Unknown table 'User'`, the simplest fix is to set `lower_case_table_names=1` in MySQL's config (`/etc/mysql/my.cnf`, section `[mysqld]`) **before** creating the database, then restart MySQL. On Windows and macOS defaults this is not an issue.

## 10. Network protocol reference

Messages are **one line of text** (terminated by a newline) with fields separated by commas. The first field is the command. Because of this, **commas must not appear inside usernames, passwords, nicknames or chat messages** (a message such as `hello, world` will be cut at the comma).

### Client → Server

| Command | Format | Meaning |
|---------|--------|---------|
| `client-verify` | `client-verify,username,password` | Log in |
| `register` | `register,username,password,nickname,avatarIndex` | Create account (and log in) |
| `offline` | `offline,userId` | Log out |
| `chat-server` | `chat-server,text` | Global chat |
| `view-friend-list` | – | Request friends list |
| `check-friend` | `check-friend,userId` | Is this user my friend? |
| `make-friend` | `make-friend,userId` | Send friend request |
| `make-friend-confirm` | `make-friend-confirm,userId` | Accept friend request |
| `get-rank-charts` | – | Top 8 leaderboard |
| `create-room` | `create-room,` or `create-room,password` | Create a room |
| `view-room-list` | – | List open rooms |
| `go-to-room` | `go-to-room,roomId,password` | Join by ID |
| `join-room` | `join-room,roomId` | Join from the room list |
| `quick-room` | – | Join any public waiting room, or create one |
| `cancel-room` | – | Cancel waiting / matchmaking |
| `duel-request` / `agree-duel` / `disagree-duel` | `…,userId` | Challenge a friend |
| `caro` | `caro,x,y` | A move |
| `win` / `lose` | `win,x,y` / `lose,` | Game result (win by 5 / lost on time) |
| `chat` | `chat,text` | In-room chat |
| `draw-request` / `draw-confirm` / `draw-refuse` | – | Draw offer flow |
| `voice-message` | `voice-message,open-mic` etc. | Voice-chat status notices |
| `left-room` | – | Leave the current room |

### Server → Client

`server-send-id`, `login-success`, `wrong-user`, `dupplicate-login`, `banned-user`, `duplicate-username`, `chat-server`, `return-friend-list`, `check-friend-response`, `return-get-rank-charts`, `room-list`, `your-created-room`, `go-to-room`, `room-fully`, `room-not-found`, `room-wrong-password`, `make-friend-request`, `duel-notice`, `disagree-duel`, `caro`, `chat`, `draw-request`, `draw-refuse`, `draw-game`, `new-game`, `competitor-time-out`, `voice-message`, `left-room`, `banned-notice`, `warning-notice`.

The two players' IPs are also exchanged in `go-to-room` so that clients can stream **voice chat directly to each other over UDP port 5555** (the server does not relay audio).

## 11. Troubleshooting

| Problem | Likely cause / fix |
|---------|--------------------|
| `Connection to database failed` + `Unknown database 'caro_game'` | You did not create the database (Step 2) or `DATABASE_NAME` in `DAO.java` differs from the real name. |
| `Access denied for user 'root'@'localhost'` | Wrong `JDBC_USER` / `JDBC_PASSWORD` in `DAO.java`. |
| `Communications link failure` / `Connection refused` | MySQL is not running, or is on a different host/port. Start MySQL and check the port (default 3306). |
| `The server time zone value '...' is unrecognized` | Add a time zone to the URL in `DAO.java`: `…/caro_game?useSSL=false&serverTimezone=UTC` |
| `Public Key Retrieval is not allowed` | Add `&allowPublicKeyRetrieval=true` to the URL in `DAO.java`. |
| `ClassNotFoundException: com.mysql.jdbc.Driver` | The JDBC jar is not on the classpath. Use the `-cp` commands above (or add `lib/mysql-connector-java-8.0.13.jar` in your IDE). |
| `Address already in use` / `BindException` on startup | Another program (or a second copy of the server) is already using port 7777. Close it and retry. |
| `'javac' is not recognized` | JDK is not installed or not on `PATH` (see 3.1). |
| Vietnamese text looks like `Ã¡`, `?` | Compile with `-encoding UTF-8`. |
| Clients cannot connect from another PC | Firewall blocking TCP 7777, or the client still points to `127.0.0.1`. |
| `Table '...banned_user' doesn't exist` (Linux) | Table-name case issue – see the note in section 9. |

## 12. Known limitations

- Server port (`7777`) and database settings are hard-coded in the source (`Server.java`, `DAO.java`).
- Passwords are stored and transmitted in **plain text**; there is no encryption (TLS) or hashing.
- The comma-separated protocol breaks if users type commas in names/passwords/chat.
- Each client thread opens its own database connection, which is fine for a classroom/demo scale but not for large numbers of players.
- The Admin panel has no login of its own – anyone who can run the server can use it.
