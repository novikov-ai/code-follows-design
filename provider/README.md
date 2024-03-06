# Логический дизайн

Логический дизайн провайдера предприятий заключается в том, что он предоставляет, по сути, CRUD для работы с сущностью `Enterprise`

Текущий функционал предполагает, что мы можем:

1. Получать все модели предприятий
2. Получить предприятия, принадлежащие конкретному менеджеру
   
... 

Начиная думать про то, как код следует из логического дизайна, сразу бросается в глаза, что присутствует какой-то непонятный метод для получения пробега по конкретному автомобилю. Как будто он должен быть вообще в другом модуле. Переносим его в другое место. 

Добавляем пред-пост условия, чтобы убедиться, что наш код соответствует логическому дизайну и не требует больше от нас правок.

# Исходный код

~~~go
package enterprises

import (
	"car-park/internal/models"
	"car-park/internal/storage"
	"context"
	"fmt"
	"github.com/gin-gonic/gin"
	"os"
	"time"
)

type Provider struct {
	db storage.Client
}

func New(st storage.Client) *Provider {
	return &Provider{
		db: st,
	}
}

func (p *Provider) FetchAll(ctx context.Context) []models.Enterprise {
	query := `SELECT * FROM enterprise`
	resp, err := p.db.Query(ctx, query)
	if err != nil {
		fmt.Fprintf(os.Stderr, "unable to proceed query: %v\n", err)
		return []models.Enterprise{}
	}

	var (
		id          int64
		title, city string
		established time.Time
		utc         int
	)

	enterprises := make([]models.Enterprise, 0)
	for resp.Next() {
		err = resp.Scan(&id, &title, &city, &established, &utc)
		if err != nil {
			fmt.Fprintf(os.Stderr, "scan failed: %v\n", err)
			return []models.Enterprise{}
		}

		enterprises = append(enterprises, models.Enterprise{
			ID:          id,
			Title:       title,
			City:        city,
			Established: established.Round(time.Second),
			UTC:         utc,
		})
	}

	return enterprises
}

func (p *Provider) FetchAllByManagerID(ctx *gin.Context, managerID int64) []models.Enterprise {
	query := `SELECT e.*
FROM enterprise as e
JOIN manager_enterprise as me on me.enterprise_id = e.id
WHERE me.manager_id = $1;
`

	resp, err := p.db.Query(ctx, query, managerID)
	if err != nil {
		fmt.Fprintf(os.Stderr, "unable to proceed query: %v\n", err)
		return []models.Enterprise{}
	}

	var (
		id          int64
		title, city string
		utc         int
		established time.Time
	)

	enterprises := make([]models.Enterprise, 0)
	for resp.Next() {
		err = resp.Scan(&id, &title, &city, &established, &utc)
		if err != nil {
			fmt.Fprintf(os.Stderr, "scan failed: %v\n", err)
			return []models.Enterprise{}
		}

		enterprises = append(enterprises, models.Enterprise{
			ID:          id,
			Title:       title,
			City:        city,
			Established: established.Round(time.Second),
			UTC:         utc,
		})
	}

	return enterprises
}

func (p *Provider) SumMileageByVehicle(ctx context.Context, vehicleID int64,
	start, end *time.Time) int64 {
	query := `SELECT SUM(tr.track_length)
FROM vehicle as v
JOIN enterprise as en ON v.enterprise_id = en.id
JOIN trip as tr ON v.id = tr.vehicle_id
WHERE en.id = $1 AND
	EXTRACT(EPOCH FROM tr.started_at) >= $2 AND
	EXTRACT(EPOCH FROM tr.started_at) <= $3;`

	if start == nil {
		zeroTime := time.Unix(0, 0)
		start = &zeroTime
	}

	if end == nil {
		now := time.Now()
		end = &now
	}

	resp, err := p.db.Query(ctx, query, vehicleID, start.Unix(), end.Unix())
	if err != nil {
		fmt.Fprintf(os.Stderr, "unable to proceed query: %v\n", err)
		return 0
	}

	var sum int64

	for resp.Next() {
		err = resp.Scan(&sum)
		if err != nil {
			fmt.Fprintf(os.Stderr, "scan failed: %v\n", err)
			return 0
		}
	}

	return sum
}
~~~

# Код после рефакторинга (код следует за логическим дизайном)

~~~go
package enterprises

import (
	"car-park/internal/models"
	"car-park/internal/storage"
	"context"
	"fmt"
	"github.com/gin-gonic/gin"
	"os"
	"time"
)

type Provider struct {
	db storage.Client
}

// Postcondition:
// Создан новый провайдер с указанным хранилищем
func New(st storage.Client) *Provider {
	return &Provider{
		db: st,
	}
}

// Postcondition:
// Из БД получены все предприятия
func (p *Provider) FetchAll(ctx context.Context) []models.Enterprise {
	query := `SELECT * FROM enterprise`
	resp, err := p.db.Query(ctx, query)
	if err != nil {
		fmt.Fprintf(os.Stderr, "unable to proceed query: %v\n", err)
		return []models.Enterprise{}
	}

	var (
		id          int64
		title, city string
		established time.Time
		utc         int
	)

	enterprises := make([]models.Enterprise, 0)
	for resp.Next() {
		err = resp.Scan(&id, &title, &city, &established, &utc)
		if err != nil {
			fmt.Fprintf(os.Stderr, "scan failed: %v\n", err)
			return []models.Enterprise{}
		}

		enterprises = append(enterprises, models.Enterprise{
			ID:          id,
			Title:       title,
			City:        city,
			Established: established.Round(time.Second),
			UTC:         utc,
		})
	}

	return enterprises
}

// Precondition:
// Менеджер существует в системе

// Postcondition:
// Получены все предприятия, принадлежащие менеджеру
func (p *Provider) FetchAllByManagerID(ctx *gin.Context, managerID int64) []models.Enterprise {
	query := `SELECT e.*
FROM enterprise as e
JOIN manager_enterprise as me on me.enterprise_id = e.id
WHERE me.manager_id = $1;
`

	resp, err := p.db.Query(ctx, query, managerID)
	if err != nil {
		fmt.Fprintf(os.Stderr, "unable to proceed query: %v\n", err)
		return []models.Enterprise{}
	}

	var (
		id          int64
		title, city string
		utc         int
		established time.Time
	)

	enterprises := make([]models.Enterprise, 0)
	for resp.Next() {
		err = resp.Scan(&id, &title, &city, &established, &utc)
		if err != nil {
			fmt.Fprintf(os.Stderr, "scan failed: %v\n", err)
			return []models.Enterprise{}
		}

		enterprises = append(enterprises, models.Enterprise{
			ID:          id,
			Title:       title,
			City:        city,
			Established: established.Round(time.Second),
			UTC:         utc,
		})
	}

	return enterprises
}
~~~