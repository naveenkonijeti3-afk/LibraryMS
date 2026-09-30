# Library Management System

A full-stack web app with two logins: **student** and **librarian**.

- **Front end:** HTML, CSS and plain JavaScript (no build step)
- **Back end:** Node.js + Express REST API with JWT login
- **SQL database:** SQLite for users, sections, books and borrowals
- **NoSQL database:** MongoDB for the activity log and student notifications

## Features

**Students**
- Sign in, search books by title, author, ISBN or section
- Borrow books (default: 14 days, up to 3 at a time)
- Return a book: **on or before the due date earns 10 reward points**
- **Late return adds a fine** (default 5 per day). Fines must be paid at the desk before borrowing again
- See due dates, overdue status, fines, history and notifications

**Librarian**
- Dashboard: titles, copies, books on loan, overdue, unpaid fines, stock by section
- Add, edit and remove books (with copy counts)
- Add and remove sections
- Add and remove students (removal is blocked while they hold books or owe fines)
- Issue books, accept returns, mark fines as paid
- Activity log of everything that happens

## Quick start

Requirements: Node.js 18+ and (optionally) MongoDB.

```bash
npm install
cp .env.example .env        # Windows: copy .env.example .env
npm run seed                # optional: sample sections, books and students
npm start
```

Open http://localhost:3000

### Demo accounts (after `npm run seed`)

| Role      | Email                     | Password       |
|-----------|---------------------------|----------------|
| Librarian | librarian@library.com     | librarian123   |
| Student   | asha@student.com          | student123     |
| Student   | ravi@student.com          | student123     |
| Student   | meera@student.com         | student123     |

The librarian account is created automatically on first start from `ADMIN_EMAIL` and `ADMIN_PASSWORD` in `.env`.
The seed data includes one overdue loan (Ravi, *Clean Code*) so you can see a fine, and a returned book that earned Asha points.

### MongoDB

The activity log and notifications use MongoDB. Pick one:

- **Local:** install MongoDB Community and leave `MONGO_URI=mongodb://127.0.0.1:27017/library`
- **Cloud (free):** create a MongoDB Atlas cluster and paste its connection string into `MONGO_URI`
- **Docker:** `docker run -d -p 27017:27017 --name mongo mongo:7`

If MongoDB is not running, the app still works. Only the activity log and notifications are switched off.

## Library rules (change in `.env`)

| Setting                 | Default | Meaning                              |
|-------------------------|---------|--------------------------------------|
| `LOAN_DAYS`             | 14      | Days until a book is due             |
| `MAX_BOOKS_PER_STUDENT` | 3       | Books a student can hold at once     |
| `FINE_PER_DAY`          | 5       | Fine for each day late               |
| `REWARD_POINTS`         | 10      | Points for an on-time return         |
| `CURRENCY`              | ₹       | Symbol shown next to fines           |

## Project structure

```
library-management-system/
├── package.json
├── .env.example
├── server/
│   ├── server.js            Express app, routes, error handler
│   ├── config.js            Reads .env
│   ├── seed.js              Sample data
│   ├── db/
│   │   ├── sql.js           SQLite schema and connection
│   │   └── mongo.js         MongoDB models (ActivityLog, Notification)
│   ├── middleware/auth.js   JWT check and role guard
│   ├── routes/
│   │   ├── auth.js          login, me
│   │   ├── books.js         book CRUD + search
│   │   ├── sections.js      section CRUD
│   │   ├── students.js      librarian: add/remove students
│   │   ├── borrowals.js     borrow, return, fines, rewards
│   │   ├── stats.js         dashboard numbers
│   │   └── misc.js          activity log + notifications (MongoDB)
│   └── utils/               date/fine helpers, safe Mongo logger
└── public/
    ├── index.html           login page
    ├── student.html         student app
    ├── librarian.html       librarian app
    ├── css/style.css
    └── js/                  common.js, login.js, student.js, librarian.js
```

## Database design

**SQL (SQLite)**

| Table       | Key columns                                                                    |
|-------------|--------------------------------------------------------------------------------|
| `users`     | id, name, email, password_hash, role (`student`/`librarian`), roll_no, reward_points |
| `sections`  | id, name, description                                                          |
| `books`     | id, title, author, isbn, section_id → sections, total_copies, available_copies |
| `borrowals` | id, user_id → users, book_id → books, borrow_date, due_date, return_date, fine, fine_paid, reward_earned, status |

Borrowing and returning run inside SQL transactions, so copy counts, rewards and fines always stay consistent.

**NoSQL (MongoDB)**

| Collection     | Fields                                            |
|----------------|---------------------------------------------------|
| `activitylogs` | action, actorId, actorName, role, details, createdAt |
| `notifications`| userId, message, type (`info`/`reward`/`fine`), read, createdAt |

## API overview

All routes except `/api/config` and `/api/auth/login` need `Authorization: Bearer <token>`.

| Method | Route                          | Who        | Purpose                        |
|--------|--------------------------------|------------|--------------------------------|
| POST   | `/api/auth/login`              | public     | Sign in (`email`, `password`, `role`) |
| GET    | `/api/auth/me`                 | any        | Current user                   |
| GET    | `/api/config`                  | public     | Library rules                  |
| GET    | `/api/books?q=&section=`       | any        | Search books                   |
| POST/PUT/DELETE | `/api/books[/:id]`    | librarian  | Manage books                   |
| GET    | `/api/sections`                | any        | Sections with counts           |
| POST/DELETE | `/api/sections[/:id]`     | librarian  | Manage sections                |
| GET/POST/DELETE | `/api/students[/:id]` | librarian  | Manage students                |
| POST   | `/api/borrowals`               | any        | Borrow (`bookId`; librarian adds `studentId`) |
| POST   | `/api/borrowals/:id/return`    | owner/librarian | Return a book             |
| GET    | `/api/borrowals/mine`          | student    | Own loans and history          |
| GET    | `/api/borrowals?status=`       | librarian  | All loans (`borrowed`, `overdue`, `returned`) |
| POST   | `/api/borrowals/:id/pay-fine`  | librarian  | Mark a fine as paid            |
| GET    | `/api/stats`                   | librarian  | Dashboard data                 |
| GET    | `/api/activity`                | librarian  | Activity log (MongoDB)         |
| GET/POST | `/api/notifications[/read]`  | any        | Notifications (MongoDB)        |

## Upload to GitHub

```bash
cd library-management-system
git init
git add .
git commit -m "Library management system"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

`.gitignore` already keeps `node_modules`, `.env` and the database file out of the repo.

## Before using this for real

- Set a long random `JWT_SECRET` and change the librarian password in `.env`
- Serve over HTTPS and restrict CORS to your own domain
- Add rate limiting on `/api/auth/login`
- Host the SQLite file on persistent storage, or move the SQL layer to MySQL/PostgreSQL

## License

MIT
