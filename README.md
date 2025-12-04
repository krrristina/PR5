# Практическая работа № 5
## Тема: "Подключение к PostgreSQL через database/sql. Выполнение простых запросов (INSERT, SELECT)"
## Подготовила: Сорокина К.С., ЭФМО-01-25.
### Выполнены следующие цели:
1.	Установить и настроить PostgreSQL локально.
2.	Подключиться к БД из Go с помощью database/sql и драйвера PostgreSQL.
3.	Выполнить параметризованные запросы INSERT и SELECT.
4.	Корректно работать с context, пулом соединений и обработкой ошибок.

### Версия psql: 
psql (PostgreSQL) 17.6

#### Команда для запуска:
```bash
go run .
```

#### Создание БД:
```bash
CREATE DATABASE todo;
```

#### Подключение к БД:
![BD](https://github.com/krrristina/PR5/blob/main/screenshots/create%20bd.png)

#### Создание таблицы задач + проверка
![table](https://github.com/krrristina/PR5/blob/main/screenshots/create%20table.png)

#### Запуск программы:
![go run](https://github.com/krrristina/PR5/blob/main/screenshots/%D0%B7%D0%B0%D0%BF%D1%83%D1%81%D0%BA%20%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D1%8B.png)

#### Проверка через PgAdmin:
![проверка](https://github.com/krrristina/PR5/blob/main/screenshots/%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%BA%D0%B0%20%D1%87%D0%B5%D1%80%D0%B5%D0%B7%20pgadmin.png)

### Фрагменты кода:
#### bd.go
```bash
package main

import (
	"context"
	"database/sql"
	"log"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func openDB(dsn string) (*sql.DB, error) {
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		return nil, err
	}

	// настройки пула --- достаточно для локалки
	db.SetMaxOpenConns(10)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(30 * time.Minute)

	// проверка соединения с таймаутом
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		return nil, err
	}

	log.Println("Connected to PostgreSQL")
	return db, nil
}
```

#### repository.go
```bash
package main

import (
	"context"
	"database/sql"
	"time"
)

// Task — модель для сканирования результатов SELECT
type Task struct {
	ID        int
	Title     string
	Done      bool
	CreatedAt time.Time
}

type Repo struct {
	DB *sql.DB
}

func NewRepo(db *sql.DB) *Repo { return &Repo{DB: db} }

// CreateTask — параметризованный INSERT с возвратом id
func (r *Repo) CreateTask(ctx context.Context, title string) (int, error) {
	var id int
	const q = `INSERT INTO tasks (title) VALUES ($1) RETURNING id;`
	err := r.DB.QueryRowContext(ctx, q, title).Scan(&id)
	return id, err
}

// ListTasks — базовый SELECT всех задач (демо для занятия)
func (r *Repo) ListTasks(ctx context.Context) ([]Task, error) {
	const q = `SELECT id, title, done, created_at FROM tasks ORDER BY id;`
	rows, err := r.DB.QueryContext(ctx, q)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var out []Task
	for rows.Next() {
		var t Task
		if err := rows.Scan(&t.ID, &t.Title, &t.Done, &t.CreatedAt); err != nil {
			return nil, err
		}
		out = append(out, t)
	}
	return out, rows.Err()
}
```

#### main.go
```bash
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	"github.com/joho/godotenv"
)

func main() {
	// .env не обязателен; если файла нет — ошибка игнорируется
	_ = godotenv.Load()

	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		// fallback — прямой DSN в коде (только для учебного стенда!)
		dsn = "postgres://postgres:YOUR_PASSWORD@localhost:5432/todo?sslmode=disable"
	}

	db, err := openDB(dsn)
	if err != nil {
		log.Fatalf("openDB error: %v", err)
	}
	defer db.Close()

	repo := NewRepo(db)

	// 1) Вставим пару задач
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	titles := []string{"Сделать ПЗ №5", "Купить кофе", "Проверить отчёты"}
	for _, title := range titles {
		id, err := repo.CreateTask(ctx, title)
		if err != nil {
			log.Fatalf("CreateTask error: %v", err)
		}
		log.Printf("Inserted task id=%d (%s)", id, title)
	}

	// 2) Прочитаем список задач
	ctxList, cancelList := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancelList()

	tasks, err := repo.ListTasks(ctxList)
	if err != nil {
		log.Fatalf("ListTasks error: %v", err)
	}

	// 3) Напечатаем
	fmt.Println("=== Tasks ===")
	for _, t := range tasks {
		fmt.Printf("#%d | %-24s | done=%-5v | %s\n",
			t.ID, t.Title, t.Done, t.CreatedAt.Format(time.RFC3339))
	}
}
```
### Ответы на контрольные вопросы:
1. Что такое пул соединений *sql.DB и зачем его настраивать?

Пул соединений — это группа переиспользуемых соединений к PostgreSQL. Каждое соединение — ресурсоемкая операция, поэтому вместо создания нового для каждого запроса мы берем готовое из пула.

Настройка:
- SetMaxOpenConns(10) — не создавать более 10 одновременных соединений (экономия ОЗУ и ресурсов серверов).
- SetMaxIdleConns(5) — держать 5 соединений в резерве для быстрого ответа на запросы.
- SetConnMaxLifetime(30m) — переиспользовать соединение 30 минут (предотвращает утечки памяти).

Правильная настройка пула повышает производительность приложения и стабильность под нагрузкой.

2. Почему используем плейсхолдеры $1, $2?
   
Плейсхолдеры защищают от SQL-инъекций — одной из самых опасных уязвимостей в веб-приложениях. При использовании плейсхолдеров значения параметров автоматически экранируются драйвером PostgreSQL, поэтому злоумышленник не может внедрить вредоносный SQL-код.

3. Query, QueryRow и Exec отличаются?

Exec — для команд, не возвращающих строки (INSERT, UPDATE, DELETE). Возвращает только информацию о том, сколько строк было затронуто.
QueryRow — для SELECT, который должен вернуть ровно одну строку. Удобнее чем Query для одной записи, не нужен цикл.
Query — для SELECT, который может вернуть много строк. Необходимо пройти по результатам циклом for rows.Next().
В нашем коде:
- CreateTask использует QueryRow (вставка + возврат id)
- ListTasks использует Query (вывод всех задач)
- CreateMany использует Exec через транзакцию
