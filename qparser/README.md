# Логический дизайн

Логический дизайн инструмента для парсинга запроса заключается в том, что он предоставляет функционал по работе с параметрами строки (их парсингу).

На выходе я получаю распаршенные значения из строки запроса: ID, дату старта и окончания и прочее.

Когда абстрагируешься на верхний уровень "над кодом", то сразу замечаешь сколько лишнего присутствует в реализации. 

Так, удалось сформулировать основное назначение данного пакета, избавиться от дублирования кода и даже от лишней зависимости `github.com/gin-gonic/gin`, что позволило сделать код легче читаемым, лучше структурированным и поддерживаемым.

# Исходный код

~~~go
package query

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"strconv"
	"time"
)

func ParseQueryParamsIdStartEndDates(c *gin.Context) (int64, *time.Time, *time.Time) {
	query := c.Request.URL.Query()
	id := query.Get("vehicle")

	idValue, err := strconv.ParseInt(id, 10, 64)
	if err != nil {
		idValue = 0
	}

	var startValue *time.Time
	start := query.Get("start")
	if start == "" {
		zeroUnix := time.Unix(0, 0)
		startValue = &zeroUnix
	} else {
		startValue = parseQueryTimeParam(start)
	}

	var endValue *time.Time
	end := query.Get("end")
	if end == "" {
		timeNow := time.Now()
		endValue = &timeNow
	} else {
		endValue = parseQueryTimeParam(end)
	}

	return idValue, startValue, endValue
}

func ParseQueryParams(c *gin.Context) (int64, *time.Time, *time.Time) {
	query := c.Request.URL.Query()
	id := query.Get("id")

	idValue, err := strconv.ParseInt(id, 10, 64)
	if err != nil {
		idValue = 0
	}

	var startValue *time.Time
	start := query.Get("start")
	if start == "" {
		zeroUnix := time.Unix(0, 0)
		startValue = &zeroUnix
	} else {
		startValue = parseQueryTimeParam(start)
	}

	var endValue *time.Time
	end := query.Get("end")
	if end == "" {
		timeNow := time.Now()
		endValue = &timeNow
	} else {
		endValue = parseQueryTimeParam(end)
	}

	return idValue, startValue, endValue
}

func ParseQueryParamsIdStartEndUnix(c *gin.Context) (int64, int64, int64) {
	query := c.Request.URL.Query()
	id := query.Get("vehicle")

	idValue, err := strconv.ParseInt(id, 10, 64)
	if err != nil {
		idValue = 0
	}

	start := query.Get("start")
	startUnix, err := strconv.ParseInt(start, 10, 64)
	if err != nil {
		startUnix = 0
	}

	startLocalTime := time.UnixMilli(startUnix)
	startUTC := startLocalTime.UTC()

	end := query.Get("end")
	endUnix, err := strconv.ParseInt(end, 10, 64)

	endLocalTime := time.UnixMilli(endUnix)
	endUTC := endLocalTime.UTC()

	if err != nil || end == "" {
		endUTC = time.Now().UTC()
	}

	return idValue, startUTC.Unix(), endUTC.Unix()
}

func GetVehicleStartEndTimeUnix(c *gin.Context) (int64, int64, int64) {
	query := c.Request.URL.Query()
	id := query.Get("vehicle")

	idValue, err := strconv.ParseInt(id, 10, 64)
	if err != nil {
		idValue = 0
	}

	start := query.Get("start")
	startUnix := parseQueryTimeParam(start).Unix()

	startLocalTime := time.Unix(startUnix, 0)
	startUTC := startLocalTime.UTC()

	end := query.Get("end")
	endUnix := parseQueryTimeParam(end).Unix()

	endLocalTime := time.Unix(endUnix, 0)
	endUTC := endLocalTime.UTC()

	if err != nil || end == "" {
		endUTC = time.Now().UTC()
	}

	return idValue, startUTC.Unix(), endUTC.Unix()
}

func parseQueryTimeParam(value string) *time.Time {
	parsed, err := time.Parse(time.RFC3339, fmt.Sprintf("%sT00:00:00.000Z", value))
	if err != nil {
		return nil
	}

	return &parsed
}
~~~

# Код после рефакторинга (код следует за логическим дизайном)

~~~go
package qparser

import (
	"errors"
	"fmt"
	"net/url"
	"strconv"
	"time"
)

const (
	idKey      = "id"
	vehicleKey = "vehicle"
	startKey   = "start"
	endKey     = "end"
)

var queryKeys = map[string]struct{}{
	idKey:      {},
	vehicleKey: {},
	startKey:   {},
	endKey:     {},
}

type ParsedValues struct {
	values map[string]string
}

func New(url *url.URL) (ParsedValues, error) {
	if url == nil {
		return ParsedValues{}, errors.New("url missed")
	}

	vv := make(map[string]string)
	query := url.Query()

	for key := range queryKeys {
		vv[key] = query.Get(key)
	}

	return ParsedValues{
		values: vv,
	}, nil
}

func (pv ParsedValues) GetID() int {
	return pv.getInt(idKey)
}

func (pv ParsedValues) GetVehicle() int64 {
	return int64(pv.getInt(vehicleKey))
}

func (pv ParsedValues) getInt(key string) int {
	value, err := strconv.Atoi(pv.values[key])
	if err != nil {
		return 0
	}

	return value
}

func (pv ParsedValues) GetStartTime() *time.Time {
	if parsed := parseQueryTimeParam(pv.values[startKey]); parsed != nil {
		return parsed
	}

	zeroUnix := time.Unix(0, 0)

	return &zeroUnix
}

func (pv ParsedValues) GetStartTimeUnix() int64 {
	unix, _ := strconv.ParseInt(pv.values[startKey], 10, 64)

	localTime := time.UnixMilli(unix)
	utc := localTime.UTC()

	return utc.Unix()
}

func (pv ParsedValues) GetEndTime() *time.Time {
	if parsed := parseQueryTimeParam(pv.values[endKey]); parsed != nil {
		return parsed
	}

	timeNow := time.Now()
	return &timeNow
}

func (pv ParsedValues) GetEndTimeUnix() int64 {
	unix, err := strconv.ParseInt(pv.values[endKey], 10, 64)

	localTime := time.UnixMilli(unix)
	utc := localTime.UTC()

	if err != nil {
		utc = time.Now().UTC()
	}

	return utc.Unix()
}

func parseQueryTimeParam(value string) *time.Time {
	if value == "" {
		return nil
	}

	parsed, err := time.Parse(time.RFC3339, fmt.Sprintf("%sT00:00:00.000Z", value))
	if err != nil {
		return nil
	}

	return &parsed
}
~~~
